# Investigação: cadernos no Tekla Structures a partir do IFC/DWG, sem redesenhar peças

Data: 2026-10-13
Fontes: documentação oficial Trimble/Tekla (User Assistance 2025 e 2026), listadas ao final.

## Conclusão

Sim, é possível. Mas a via correta não é o DWG: é a conversão do IFC em objetos nativos do Tekla.
O DWG dentro do desenho é só gráfico 2D. Não gera marca de peça, numeração, lista de material nem cotas associativas.
O caderno do Tekla é gerado a partir de objetos do modelo. Se as peças existem como objetos nativos, cada caderno novo é
uma questão de template de desenho (cloning template) e propriedades de desenho, não de redesenho.

## As três vias avaliadas

| Via | O que vira dentro do Tekla | Serve para caderno de fabricação? | Esforço por caderno novo |
|---|---|---|---|
| 1. IFC convertido em objetos nativos (Manage > Convert IFC objects) | Partes com perfil (beam/column/plate) ou items (sólido 3D) | Sim. Marcas, numeração, cotas associativas, relatórios, GA, assembly e single-part | Baixo: template + propriedades de desenho |
| 2. IFC como modelo de referência exibido no desenho | Objeto de referência (wireframe ou outline) | Parcial. Aparece no GA, assembly e cast unit. Aceita nota associativa. Não tem marca de peça nem numeração | Médio: anotação manual em cada caderno |
| 3. DWG linkado no desenho (aba Drawing > DWG/DXF ou 2D Library) | Gráfico 2D vinculado ao arquivo | Não. Sem ID de peça, sem marca, sem BOM, sem clash | Alto: equivale ao polígono atual |

Via 3 é exatamente o problema que vocês querem resolver, só com outro formato. Descartar.

## Via 1 em detalhe: o que a documentação garante

**Tipos IFC suportados na conversão:** IfcBeam, IfcColumn, IfcMember, IfcPile, IfcFooting, IfcPlate, IfcDiscreteAccessory,
IfcSlab, IfcWall, IfcWallStandardCase, IfcRailing, IfcBuildingElementPart.
Montantes, guias, bloqueadores e ombreiras de Steel Frame normalmente saem como IfcMember ou IfcBeam. Cobertos.

**Dois resultados possíveis:**
- Extrusão (part): perfil extrudado no comprimento. É o que vocês querem para perfil galvanizado C/U. Mantém perfil,
  material, comprimento. Gera marca e cota como qualquer viga nativa.
- Item: sólido 3D (B-rep) no shape catalog, classe 996. Ainda é objeto nativo, entra em desenho e numeração, mas não
  carrega perfil paramétrico. Funciona para peças especiais, pior para perfil padrão.

**Condição para sair como extrusão com perfil nativo:**
- Tekla 2025 e anteriores: o IFC precisa ser IFC2x3 Coordination View 2.0. IFC4 não converte em 2025 e anteriores.
- Tekla 2026: a conversão passou a aceitar IFC4 (release note "Enhanced IFC interoperability").
- Perfis paramétricos I, L, U, C, T, Z, retangular e circular têm casamento por nome ou por dimensão.
  Perfil C/U de chapa dobrada é paramétrico, então entra nesse grupo. Nome do perfil no IFC deve bater com o catálogo
  Tekla ou ser mapeado (botão Check > mapeamento de perfis e materiais, arquivos MappedProfiles-default.txt).

**Limitações documentadas:**
- Parafusos, armadura e solda não convertem.
- Offset de perfil não é suportado. Chanfros podem sair errados.
- Perfil pode sair rotacionado, porque o IFC não carrega orientação paramétrica. Precisa conferência.
- Objeto com múltiplas representações não converte.
- Qualidade do resultado depende da qualidade do IFC de origem.

**Gestão de revisão:** o Tekla mantém o vínculo IFC > objeto nativo. Em nova revisão do IFC, a ferramenta de
conversion change management lista New / Changed / Deleted / Up-to-date e atualiza só o que mudou. Isso significa
que o modelo nativo acompanha o IFC sem reconverter tudo.

## Fluxo proposto

1. Confirmar a origem do IFC. Se o IFC foi exportado do próprio Tekla, o modelo nativo já existe: não há o que
   converter, o caderno sai direto desse modelo. Se veio de outro software (Vertex BD, FrameCAD, Revit, Archicad etc.),
   exportar como IFC2x3 CV 2.0 (ou usar Tekla 2026 com IFC4).
2. Inserir o IFC como reference model. Rodar Check na caixa IFC object conversion. Mapear perfis e materiais faltantes
   uma única vez. Esse mapeamento fica salvo no modelo e pode ser copiado para a pasta de firma.
3. Converter com "Conversion as extrusion" para os perfis galvanizados. Deixar B-rep como item só para peças especiais.
4. Auditar amostra: rotação de perfil, comprimento, classe (990 = perfil paramétrico nomeado, mais confiável; 995/996 =
   B-rep sem nome, revisar).
5. Numerar. Montar 1 cloning template por tipo de caderno (painel de parede, piso, cobertura). Gerar os demais por clone.
6. Em revisão de projeto, usar o change management do IFC em vez de reconverter.

## Riscos e incertezas explícitas

- Não encontrei na documentação confirmação de que cotas em objetos de referência (via 2) sejam associativas. Notas
  associativas sim. Tratar via 2 como visualização, não como caderno de fabricação.
- A conversão como item não gera dados de perfil para máquina (rollforming) nem para relatório de perfil. Se o
  objetivo inclui link com máquina, só a via "extrusão" serve.
- Se o IFC de origem não nomeia os perfis conforme catálogo, o mapeamento inicial é trabalho manual, mas feito 1 vez.
- Teste de prova recomendado: 1 painel, cerca de 30 a 50 peças, antes de aplicar ao modelo completo.

## Fontes

- Convert IFC objects into native Tekla Structures objects (2025): https://support.tekla.com/doc/tekla-structures/2025/int_conversion_management
- Convert IFC objects (2026, inclui IFC4): https://support.tekla.com/doc/tekla-structures/2026/int_conversion_management
- Enhanced IFC interoperability, release 2026: https://support.tekla.com/doc/tekla-structures/2026/rel_int_enhanced_ifc_interoperability
- Show reference objects in drawings (2025): https://support.tekla.com/doc/tekla-structures/2025/dra_setting_visibility_and_appearance_of_reference_models
- DWG and DXF (2025): https://support.tekla.com/doc/tekla-structures/2025/int_dwg_and_dxf
- Import 2D or 3D DWG or DXF files (2025): https://support.tekla.com/doc/tekla-structures/2025/int_importing_dwg_dxf_files
- Create items (2025): https://support.tekla.com/doc/tekla-structures/2025/mod_items
- Drawing cloning improvements (2021): https://support.tekla.com/fi/doc/tekla-structures/2021/rel_2021_dra_cloning_improvements
- Create drawings with cloning templates (2021): https://support.tekla.com/sv/doc/tekla-structures/2021/dra_creating_drawings_with_cloning_templates
