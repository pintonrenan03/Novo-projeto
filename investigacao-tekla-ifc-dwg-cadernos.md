# Investigação: cadernos da escada sem redesenhar peças a cada caderno

Data: 2026-10-13. Revisão 2, após leitura das sessões "Cadernos de fabricação e montagem", "Continuidade do chat anterior"
e "Continuidade da sessão anterior" (projeto Conexão Claude Tekla, HANDOFF_2026-10-12 e resumos de fim de sessão).
Fontes externas: documentação Trimble (Tekla User Assistance 2025/2026 e Tekla Open API 2024 a 2026), listadas ao final.

## 1. Contexto lido no projeto

- Modelo: `ESCADA NOVO AIRÃO.db1` no Tekla 2026, 1732 peças, escada C1 Novo Airão, obra em Steel Frame.
- Ponte: `TeklaBridge.exe 0.4.0` (C# net48, Tekla Open API) <-> Claude Code por HTTP em localhost:8765. Hoje só cobre
  o modelo (peças, psets, furos, lotes de escrita). Não cobre o namespace de desenhos.
- IFC: `ifc/ESCADA_NOVO_AIRAO_2026-10-12.ifc`, IFC4X3 Reference View, exportado pelo diálogo do Tekla. 318 peças,
  58 conjuntos com marca, 120 grupos de parafusos, 352 furos, 0 soldas.
- Pipeline dos cadernos: `cadernos/pipeline/` em Python 3.12 (ifcopenshell, trimesh, scipy, shapely, ezdxf,
  matplotlib). `base.py` faz vistas, HLR, cores, carimbo, PDF+PNG e checagem de moldura/sobreposição. `planif.py` faz
  a planificação das chapas dobradas. `fab.py`, `mont.py`/`mont2.py` e `inst.py` montam cada folha a partir das
  malhas do IFC, com vistas, cotas, balões e detalhes escritos por marca e por conjunto.
- Estado em 2026-10-13: fabricação (19 folhas A4) e montagem (9 folhas A3) geradas, R02, aguardando revisão.
  Instalação pausada e não aprovada: problema de conteúdo dos detalhes, não de layout.

Leitura do problema: o "polígono" é a malha do IFC projetada em 2D pelo pipeline. Cada caderno novo exige escrever de
novo, em Python, vistas, cotas e detalhes por peça. O trabalho pesado não é traçar o contorno (isso já é automático),
é compor vista, cota, balão e seção. Um DWG puro não elimina essa composição. O que elimina é usar o motor de desenho
do próprio Tekla, que já faz vista, linha oculta, seção, cota automática, marca de peça e planificação.

## 2. Resposta direta

Sim, é possível parar de reconstruir as peças a cada caderno. A via é o Tekla gerar os desenhos (single-part, assembly
e GA) a partir do modelo que já existe, por meio da Open API na ponte, e o pipeline Python passar a compor cadernos em
cima do que o Tekla entrega (PDF ou DWG), em vez de recriar cada vista a partir da malha do IFC.

O DWG aparece como formato de saída do Tekla, não como fonte primária. DWG 2D não carrega identidade de peça
(sem GUID, sem marca por objeto). O IFC continua sendo a fonte de dados (marcas, pesos, furos), e o desenho do Tekla
passa a ser a fonte de geometria 2D.

## 3. Vias avaliadas

| Via | O que o Tekla faz | O que o pipeline faz | Esforço por caderno novo | Risco |
|---|---|---|---|---|
| A. Atual: IFC -> malha -> polígono -> folha | Só exporta IFC | Tudo: projeção, HLR, seção, cota, balão, planificação, layout | Alto: código por peça e por conjunto | Já conhecido (instalação reprovada por conteúdo) |
| B. Tekla gera desenhos pela Open API, imprime PDF | Vista, linha oculta, seção, cota automática, marca, planificação, lista de material | Lote de criação, atributos por tipo de folha, pós-processamento (capa, índice, folhas 3D) | Baixo: 1 arquivo de atributos por tipo de caderno, reutilizável em projeto novo | Depende de como as chapas dobradas foram modeladas (ver 5) |
| C. Tekla gera desenhos e exporta DWG/DXF; pipeline lê o DXF (ezdxf já está instalado) e compõe o caderno | Igual a B até o DWG | Lê blocos/camadas, posiciona nas folhas Ecológica, adiciona notas e 3D colorido | Médio: parser DXF + regras de camada | Exportação DWG não tem método direto na Open API (ver 4) |
| D. DWG 3D do modelo como fonte, no lugar do IFC | Exporta geometria | Mesmo que A, com fonte pior | Igual a A | Perde GUID, marca e psets. Descartar |

Recomendação: B para fabricação e montagem. C só se o layout Ecológica precisar dominar a folha (A1 de instalação).
A continua para a folha 1 de instalação (3D colorido por conjunto, padrão das pranchas 201 a 214), onde o Tekla rende pouco.

## 4. O que a Open API garante (documentação 2024 a 2026)

- Criação de desenhos: `SinglePartDrawing(Identifier, string atributos)`, `AssemblyDrawing`, `GADrawing`, cada um com
  `Insert()`. As vistas são criadas conforme as regras de "View creation" do arquivo de atributos padrão. Só insere
  quando não há desenho ativo (`CannotInsertDrawingException`).
- `DrawingHandler`: `GetDrawings()`, `SetActiveDrawing`, `SaveActiveDrawing`, `CloseActiveDrawing`, `UpdateDrawing`
  (exige numeração feita antes), `GetModelObjectIdentifiers()`.
- Impressão: `PrintDrawing(Drawing, DPMPrinterAttributes, string arquivo)` e `PrintDrawings(IEnumerable<Drawing>,
  DPMPrinterAttributes, string pasta)`. `DPMPrinterAttributes.OutputFileName` vale para PDF. Retorna false em falha.
- Objetos de desenho pela API: vistas, cotas (`StraightDimensionSet`), marcas, textos, linhas, símbolos. Os exemplos
  oficiais cobrem GA com várias vistas, cotas retas em vigas, marca com texto em peça, impressão.
- Exportação DWG/DXF: não existe método na Open API. Existe pelo Document manager (lote) ou por macro gravado e
  executado via `Operation.RunMacro` (prática comum, não verificada na documentação desta rodada).
- Exportação DWG/DXF (diálogo): baseada em objeto, não em linha. Gráficos agrupados em blocos por objeto; opção de
  desagrupar. Regras de camada por tipo de objeto, parte da apresentação (contorno, hachura, texto, moldura de marca)
  e filtro de seleção. Opção "snapshot to CAD model space" exporta em escala no model space. Nenhuma garantia de
  marca de peça no nome da camada ou do bloco.
- Planificação: single-part drawing com "Unfolded = Yes" nas View creation attributes planifica polybeam e chapa
  dobrada. Cotas de dobra dependem de `XS_DRAW_BENDING_LINE_DIMENSIONS_IN_UNFOLDING`. Só polybeam criado pelo comando
  Polybeam planifica, e em um único plano. Para chapas feitas com "Attach to part", a extensão Plate Work Tools
  (Unfolded bent plate drawing) cobre.

## 5. Pontos a verificar no modelo antes de decidir

1. Como as chapas dobradas (d1, p5, pi1, co1, co2) estão modeladas: chapa dobrada do Tekla (polybeam/bent plate) ou
   contour plates coladas. Só a primeira planifica nativamente. Hoje a planificação é do `planif.py`, com peso batendo
   em 0,3% com o Tekla. Se for contour plate, ou remodela como bent plate, ou mantém o `planif.py` só para essas marcas.
2. Soldas: 0 no IFC e premissa no caderno (perna 3). No desenho do Tekla, solda só aparece se modelada. Decidir se
   modela as soldas (ganha símbolo automático) ou continua como nota.
3. Numeração: o desenho do Tekla usa a marca da numeração. As marcas atuais (z1, c1, ZANCA1) precisam bater com os
   prefixos de numeração do modelo.
4. Ponte: adicionar referência a `Tekla.Structures.Drawing` no `TeklaBridge` e expor 4 endpoints: criar desenho por
   identificador e arquivo de atributos, listar, atualizar, imprimir PDF. Mesma política de lote e dry_run já usada.

## 6. Teste de prova proposto

Escopo: 1 marca de perfil (z1), 1 chapa dobrada (d1) e 1 conjunto (DEGRAU2). Cerca de 3 desenhos, 1 tarde.

1. Criar 2 arquivos de atributos de desenho no Tekla: `eco_fab` (single-part, A4, Unfolded = Yes, cota acumulada da
   ponta 0, cores por tipo de objeto) e `eco_mont` (assembly, A3, lista de material, balões).
2. Pela ponte, inserir os 3 desenhos e imprimir PDF.
3. Comparar com FAB_z1, FAB_d1 e MONT_DEGRAU2 do pipeline atual, item a item do checklist de fabricação (12 itens).
4. Se passar: pipeline vira orquestrador (lote por marca, PDF do Tekla + capa, índice e folhas 3D do Python).
   Se reprovar em planificação: via B só para perfis e conjuntos, `planif.py` segue para chapas.

## 7. Incertezas explícitas

- Não li o `HANDOFF_2026-10-13.md` inteiro (só a seção 8 pelo resumo da sessão). Pode haver decisão posterior que
  altere o item 5.
- Não verifiquei na documentação se o Open API 2026 expõe exportação DWG. A busca só encontrou PDF por
  `PrintDrawing`. Tratar DWG via API como não disponível até prova em contrário.
- Cotas em objetos de referência (IFC inserido como referência num desenho) não têm associatividade confirmada na
  documentação. Por isso a via "IFC como referência no desenho" foi descartada para caderno de fabricação.

## Fontes

- SinglePartDrawing Class (2026): https://developer.tekla.com/doc/tekla-structures/2026/single-part-drawing-class-68986
- SinglePartDrawing.Insert (2026): https://developer.tekla.com/doc/tekla-structures/2026/insert-method-68996
- AssemblyDrawing.Insert (2025): https://developer.tekla.com/doc/tekla-structures/2025/insert-method-47999
- DrawingHandler Class (2026): https://developer.tekla.com/doc/tekla-structures/2026/drawing-handler-class-67403
- DrawingHandler.PrintDrawing (2025): https://developer.tekla.com/doc/tekla-structures/2025/print-drawing-method-drawing-dpm-printer-attributes-string-48573
- DPMPrinterAttributes Class (2026): https://developer.tekla.com/doc/tekla-structures/2026/dpm-printer-attributes-class-67336
- Drawing code examples: https://developer.tekla.com/documentation/drawing-code-examples
- Export a drawing in 2D DWG, DXF, or DGN (2025): https://support.tekla.com/doc/tekla-structures/2025/int_export_to_dwg_or_dxf
- Unfold polybeams in drawings (2025): https://support.tekla.com/doc/tekla-structures/2025/dra_unfolding_polybeams
- Unfolded bent plate (Plate Work Tools): https://support.tekla.com/help/tekla-structures/not-version-specific/unfolded_bent_plate
- Convert IFC objects into native objects (2026): https://support.tekla.com/doc/tekla-structures/2026/int_conversion_management
- Show reference objects in drawings (2025): https://support.tekla.com/doc/tekla-structures/2025/dra_setting_visibility_and_appearance_of_reference_models
- DWG and DXF (2025): https://support.tekla.com/doc/tekla-structures/2025/int_dwg_and_dxf
