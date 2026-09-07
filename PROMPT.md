# Prompt — Pipeline de Extração e Harmonização de Dados
## PINTEC Semestral (IBGE) + BADEPI (INPI)

> **Como usar:** entregue este prompt a uma IA juntamente com acesso aos arquivos brutos das duas bases. A IA deve executar o pipeline descrito e produzir painéis tidy prontos para análise. O tema do estudo é **livre** (Indústria 4.0, ESG, produtividade, comércio, trabalho, ou qualquer outro) — o pipeline é **agnóstico ao tema**. As únicas exigências são: usar as mesmas fontes (PINTEC Semestral e/ou BADEPI) e seguir as regras abaixo.

---

## 1. Seu papel e objetivo

Você é um engenheiro de dados responsável por construir o pipeline de preparação de dados de um estudo empírico que utiliza como fontes:

1. **PINTEC Semestral (IBGE)** — pesquisa de inovação com quebra por **CNAE 2.0 divisão** (sem quebra por UF), em ondas anuais/semestrais;
2. **BADEPI (INPI)** — base de pedidos de patentes depositados no INPI, com classificação IPC e campo tecnológico.

Seu objetivo: produzir **painéis tidy na unidade "CNAE 2.0 divisão × período"**, reprodutíveis, auditáveis e **temporalmente flexíveis**, a partir dos quais qualquer análise posterior (descritiva, índices compostos, regressões) possa ser feita.

Você **não** deve assumir o tema do estudo, as variáveis de interesse exatas, nem os períodos disponíveis — tudo isso deve ser **descoberto dinamicamente** dos dados brutos e confirmado com o usuário quando necessário.

---

## 2. Princípios universais (obrigatórios, sem exceção)

1. **Nunca faça hardcode de períodos.** Não presuma que os dados cobrem 2021–2024 (PINTEC) ou 2000–2024 (BADEPI). Novas ondas, releases e até pesquisas sucessoras podem ampliar a cobertura. Descubra períodos por inspeção de arquivos e colunas de ano, e derive todos os parâmetros temporais dessa descoberta.
2. **Inventário antes de processar.** Antes de ler qualquer dado, liste TODOS os arquivos brutos (nome, tamanho, data) e TODAS as abas dos XLSX / colunas dos CSV. Imprima o inventário no log.
3. **Reprodutibilidade total.** Todo o trabalho deve ser feito em scripts R numerados, re-executáveis de ponta a ponta, sem estado manual entre etapas. Cada execução deve gerar o mesmo resultado.
4. **Nunca modifique os dados brutos.** Leia de `data/raw/` e escreva somente em `data/processed/`, `outputs/`. Nenhum script pode alterar os originais.
5. **Auditoria explícita.** Cada etapa deve imprimir no log: contagens de linhas antes/depois de cada filtro, número de NAs, períodos detectados, divisões CNAE detectadas, e alertas sobre estruturas inesperadas.
6. **Diante do inesperado, reporte — não adivinhe.** Se uma aba, coluna ou padrão não corresponder ao esperado neste documento, interrompa e reporte ao usuário em vez de improvisar silenciosamente. Registre a divergência no log.
7. **Flexibilidade de unidade.** O padrão deste pipeline é CNAE divisão × ano. Se o usuário solicitar outra granularidade (agrupamento de divisões, seção C consolidada, etc.), adapta-se nos merges — mas a extração bruta permanece por divisão.

---

## 3. Fonte 1 — PINTEC Semestral (IBGE)

### 3.1 O que é e onde está

- Arquivos XLSX, tipicamente organizados em `data/raw/pintec/<ano>/` com subpastas por família de indicadores (ex.: `Indicadores_tematicos/`, `Indicadores_basicos/`).
- Cada pasta contém um XLSX com quebra CNAE (identifique pelo nome do arquivo conter "CNAE"; pode haver outros com quebra por porte/UF — **use apenas os de CNAE**).
- A quebra geográfica disponível é **apenas por CNAE divisão** (nacional). Não existe UF na PINTEC Semestral.

### 3.2 Estrutura interna das planilhas (regras críticas)

Estas regras valem para **todas** as abas e **todos** os anos — verifique sempre, pois há pequenas variações entre ondas:

| Elemento | Regra |
|---|---|
| **Cabeçalho** | Ocupa as **5 primeiras linhas**. Os dados começam na **linha 6**. |
| **Identificação do setor** | Coluna 1 (V1) contém o nome do setor; a numeração CNAE pode não aparecer como código explícito — o mapeamento é **por nome** (ver 3.4). |
| **Linhas de subtotal** | Existem linhas de total que devem ser **descartadas**: `Total Indústria`, `Indústrias extrativas`, `Indústrias de transformação` (e variantes). Filtre por regex tipo `^Total Ind`, `^Industrias extra`, `^Industrias de trans`, `^Fonte`. |
| **Quebra de nomes longos** | Nomes de CNAE longos **quebram em duas linhas físicas**: a segunda linha tem V2 (valor) `NA` e só continuação do texto em V1. **Regra:** se V2 não é numérico, a linha é continuação — concatene V1 com a linha anterior e descarte a linha de continuação. |
| **Linhas válidas** | Uma linha de dados válida tem V1 não-NA **e** V2 numérico. |

### 3.3 Nomenclatura de abas (variação entre ondas!)

- **Onda 2021** (primeira onda): abas sem ponto — `tab1`, `tab2`, ...
- **Ondas 2022+**: abas com ponto — `tab1.1`, `tab1.2`, `tab1.6`, `tab1.7`, `tab1.10`, `tab1.11`, ...

**Regra de robustez:** sempre busque a aba desejada pelo nome exato primeiro (ex.: `tab1.6`); se não existir, tente a variante equivalente (`tab1.6` → `tab6`; `tab1.1` → `tab1`). Use `readxl::excel_sheets()` para listar as abas disponíveis e mapear dinamicamente. Se **nenhuma** variante existir, registre no log e continue (a variável ficará ausente naquele período).

**Referência de abas (as mais usadas; confirme em cada onda):**

| Aba | Conteúdo típico | Layout de colunas (referência) |
|---|---|---|
| `tab1.2` (Temáticos) | Adoção de tecnologias digitais | V1=nome setor, V2=Total empresas, V3=Total usou alguma tecnologia, V4..V15 = **pares (Utilizou, Não utilizou)** por tecnologia (ex. 4.0: Big Data, Cloud, IA, IoT, Manufatura Aditiva, Robótica) |
| `tab1.1` (Básicos) | Inovação | V2=total empresas, V3=empresas inovadoras ativas, V4=produto/processo, V5=produto, V6=processo |
| `tab1.6` (Básicos) | Cooperação | V2=total, V3=inovadoras, V4=total cooperação |
| `tab1.7` (Básicos) | P&D | V2=total, V3=inovadoras, V4=realizou P&D interno |
| `tab1.10` (Básicos) | Financiamento/apoio público | V2=total, V3=inovadoras, V4=utilizou apoio |
| `tab1.11` (Básicos) | ESG/sustentabilidade | V2=total, V3=publicou relatório de sustentabilidade |

> ⚠️ O layout de colunas acima é **referência histórica** — ao processar uma nova onda, **confirme as posições** inspecionando o cabeçalho (linhas 1–5) e os primeiros registros antes de extrair. Se divergir, reporte.

### 3.4 Mapeamento nome do setor → código CNAE divisão

O código do CNAE pode não estar explícito; use **matching por palavras-chave** sobre o nome (V1), após normalizar (minúsculas, sem acentos). Referência de padrões para as divisões de manufatura (seção C, divisões 05–33 — confirme cobertura real de cada onda):

```
"05-09" = "extrativ|miner"          "20" = "quimic"
"10"    = "aliment"                 "21" = "farmoquim|farmaceut"
"11"    = "bebida"                  "22" = "borracha|plastico"
"12"    = "fumo"                    "23" = "minerais nao|nao-metal"
"13"    = "texte"                   "24" = "metalurg"
"14"    = "vestuar|confecc"         "25" = "produtos de metal"
"15"    = "couro|calcad|viagem"     "26" = "informatica|eletronic|optico"
"16"    = "madeira"                 "27" = "materiais eletr"
"17"    = "celulose|papel"          "28" = "maquinas e equip|maquinas, aparelhos"
"18"    = "impress|reprodu"         "29" = "veiculos automat|reboques|carrocerias"
"19"    = "coque|petroleo|biocombust" "30" = "outros equipamentos de transporte"
                                    "31" = "moveis"
                                    "32" = "produtos diversos"
                                    "33" = "manutencao|reparacao|instalacao"
```

- Atribua o **primeiro** padrão que casar (ordem importa); linhas sem match de CNAE devem ser descartadas e contadas no log.
- Após extração, **dedulique** por `(cnae, ano)` mantendo a primeira ocorrência — `[ , .SD[1], by = .(cnae, ano)]` — pois a quebra de nomes pode gerar duplicatas residuais.
- **Flexibilidade futuro:** novas ondas podem incluir divisões novas ou agregações diferentes (ex.: "05-09" pode aparecer desdobrado em 05, 06, 07...). Detecte e reporte; só incorpora com confirmação do usuário.

### 3.5 Conversão de indicadores

- Os XLSX trazem **contagens** (nº de empresas), não proporções. Converta sempre para proporção dividindo pelo denominador correto da tabela (ex.: `pct = V_utilizou / V_total_empresas`; em tabelas de básicos com população "inovadoras", o denominador pode ser V3 — verifique o cabeçalho da onda).
- Preserve também o `n_empresas` (denominador) no painel — é essencial para ponderações e para QA.
- Renomeie variáveis em `snake_case` com prefixo semântico (`pct_`, `n_`, `pilar_`).

---

## 4. Fonte 2 — BADEPI (INPI)

### 4.1 O que é e onde está

- Conjunto de CSVs (~1 GB no release v11) em `data/raw/inpi_badepi/`. O release traz **8 tabelas**; as quatro essenciais para agregação setorial têm nomes típicos (confirme no inventário):

| Tabela | Papel | Colunas essenciais |
|---|---|---|
| `*_ptn_deposito.csv` | Um registro por pedido | `NO_PEDIDO`, `ANO` (ano do depósito) |
| `*_ptn_depositante.csv` | Depositantes do pedido | `NO_PEDIDO`, `CD_PAIS_PFPJ` (país), `CD_UF_PFPJ` (UF) |
| `*_ipc_campo_tec.csv` | Classes IPC por pedido | `NO_PEDIDO`, `CD_CLASSIF` (código IPC), `CAMPO_TEC` (campo tecnológico) |
| `*_ptn_pct.csv` | Pedidos com rota PCT | `NO_PEDIDO` |

### 4.2 Parâmetros de leitura (críticos)

- `data.table::fread(..., encoding = "Latin-1", sep = ";")` — **encoding "Latin-1" com hífen**; qualquer outra grafia falha silenciosamente ou quebra a leitura de acentos.
- Use `select=` para carregar **somente** as colunas essenciais (os CSVs são grandes).
- `NO_PEDIDO` sempre como **character** (evita perda de zeros à esquerda); `ANO` como integer.

### 4.3 Fluxo de processamento (ordem obrigatória)

1. **Filtro de residência:** mantenha pedidos com ao menos um depositante `CD_PAIS_PFPJ == "BR"` (após `toupper(trimws())`). Registre os descartados.
2. **Filtro temático por IPC (opcional, conforme o estudo):** marque um pedido como "da tecnologia X" se **qualquer** classe `CD_CLASSIF` casar com os **prefixos IPC** definidos no setup do estudo (ex. para 4.0: `IA: ^G06N|^G06F18|^G06V`, `IoT/redes: ^H04L|^H04W|^H04M`, `robótica: ^B25J|^G05B`, `manufatura aditiva: ^B33Y`, `sensores: ^G01D|^G01P|^H01L`, `computação: ^G06F`). Se o estudo não exigir filtro temático, use todas as patentes.
3. **Mapeamento CAMPO_TEC → CNAE divisão:** o campo tecnológico da INPI (código 1–32) é mapeado para a divisão CNAE mais próxima via tabela de correspondência definida no `00_setup.R` do projeto (baseada em NICE/INPI + compatibilidade IPC↔ISIC/OCDE). Exemplos: 1–2→20 (Química), 3→23, 4→22, 5→29, 9→28, 10→27, 11→26, 12→31, 13→10, 16→21, 29→24. **Ao agregá-la por pedido, use o primeiro `CAMPO_TEC` mapeável** (`cnae[!is.na(cnae)][1]` por `NO_PEDIDO`); pedidos sem mapeamento são descartados e contados.
4. **Junções:** todas por `NO_PEDIDO`. `deposito(ANO) ⨝ depositante(BR) ⨝ ipc_agg(pat_tema, cnae) ⨝ pct(via_pct)`.
5. **Agregação:** por `(cnae, ano)` produza: `n_pat_total`, `n_pat_tema` (ex. `n_pat_40`), `n_pat_pct_tema` (via PCT).
6. **RTA (Revealed Technological Advantage), por ano:**
   `RTA_cnae,ano = (pat_tema_cnae / pat_total_cnae) / (pat_tema_BR / pat_total_BR)`
   calculado **dentro de cada ano**, com guarda contra divisão por zero (`pmax(x,1)`) e `Inf → NA`.
7. **Janela temporal:** filtre `ano` pela janela do estudo — mas **derivada de parâmetro**, nunca literal. Alinhe com a menor cobertura entre as fontes quando o painel exigir merge.

### 4.4 Limites conhecidos (documente no relatório)

- BADEPI cobre **apenas depósitos no INPI**; invenções brasileiras depositadas exclusivamente no exterior ficam subestimadas (a rota PCT mitiga parcialmente).
- O mapeamento CAMPO_TEC→CNAE envolve **aproximações** (um campo tecnológico pode servir a múltiplas divisões) — trate como proxy e declare a limitação.
- Divisões sem patentes mapeadas têm pilar de patentes ausente (NA) — decisão de tratamento (excluir, zerar, ou NA) deve ser explícita e documentada.

---

## 5. Harmonização e painel final

1. **Merge** PINTEC + BADEPI por `(cnae, ano)` com `all.x = TRUE` (PINTEC é a espinha dorsal; BADEPI enriquece).
2. Painel esperado: uma linha por **CNAE divisão × ano**, em `data/processed/painel_cnae_ano.rds`.
3. Variáveis-chave de saída (nomenclatura exemplo):
   - Identificação: `cnae`, `cnae_nome`, `ano`, `n_empresas`
   - PINTEC: `pct_*` (tecnologias, inovação, P&D, cooperação, apoio, ESG)
   - BADEPI: `n_pat_total`, `n_pat_40`, `rta_40`, `via_pct`
4. **Coberturas assimétricas são normais:** indicadores que só existem a partir de certa onda (ex.: temáticos 4.0 só de 2022 em diante; IA/robótica inexistentes em 2021) devem aparecer como colunas com NA nos períodos anteriores — não descarte os períodos, não preencha com zero. Documente no log qual variável começa em qual período.
5. **Exemplo de camada analítica opcional (índice composto)** — se o estudo pedir, construa pilares normalizados **min-max dentro de cada ano** (`norm_ano`: `(x - min)/(max - min)` por ano; se amplitude zero → 0.5), combine por média aritmética (ou pesos definidos pelo usuário), gere índice composto e uma **versão PCA** (1º componente, sinal alinhado com a média simples) para robustez. Mas lembre: a extração/harmonização NÃO depende dessa camada.

---

## 6. Flexibilidade temporal (seção crítica)

**Teclas do futuro:** novas ondas da PINTEC (ou de pesquisa sucessora), novos releases BADEPI, e janelas de análise diferentes devem funcionar **sem alterar código**.

Regras:

1. Todos os loops de período iteram sobre a **lista descoberta** de ondas (pastas em `data/raw/pintec/<ano>/`), nunca sobre um vetor fixo.
2. O parâmetro `anos_pintec_tema` (ondas com temáticos) deve ser detectado pela **presença da subpasta/aba temática**, não assumido.
3. Períodos BADEPI derivam do `range(ANO)` lido dos dados, filtrado por parâmetro de janela do estudo (`anos_badepi`), que por sua vez pode ser definido como "interseção com PINTEC" ou explícito pelo usuário.
4. `ano_base` (para cross-section) = max(ano) disponível na fonte relevante; nunca literal.
5. Se aparecer uma onda com estrutura desconhecida (nova pesquisa, novas abas, separador diferente): **reporte e pare** — o usuário decide o mapeamento.
6. Toda table/figura gerada deve rotular períodos dinamicamente (ex.: "média {min}–{max}"), nunca um rótulo fixo.

---

## 7. Armadilhas técnicas (Windows / OneDrive / R) — decore isto

| Problema | Sintoma | Solução |
|---|---|---|
| **BOM em arquivos R** | `Erro: invalid token in "\ufeff"` ao rodar script | PowerShell `Set-Content`/`Out-File` insere BOM. Grave sempre sem BOM: `[System.IO.File]::WriteAllText($f, $content, (New-Object System.Text.UTF8Encoding($false)))`. Se o BOM já existir, remova os 3 primeiros bytes (239,187,191). |
| **Encoding CSV BADEPI** | Acentos corrompidos / erro de parse | `fread(..., encoding = "Latin-1")` — com hífen, exatamente assim. |
| **Caminhos OneDrive com acento** | pandoc/rmarkdown falham; conversões MD→DOCX quebram | Copie o arquivo para um diretório curto sem acento (ex.: `C:\Users\<user>\AppData\Local\Temp\opencode\`), processe lá, e copie o resultado de volta. |
| **Rscript fora do PATH** | comando não encontrado | Caminho típico: `C:\Program Files\R\R-4.4.1\bin\x64\Rscript.exe` (verifique a versão instalada). |
| **Expand-Archive .docx** | "não é um formato de arquivo morto com suporte" | Copie para `.zip` antes de expandir. |
| **XLSX grandes** | leitura lenta | Leia com `readxl::read_excel`, uma aba por vez, com `.name_repair = "minimal"`; renomeie colunas para `V1..Vn` e saiba que o layout mapeia por posição. |
| **Pacotes ausentes** | erro em library() | Instale com `install.packages(pkg, type = "binary")`; verifique antes com `requireNamespace()`. |

Pacotes utilizados no pipeline de referência: `here`, `data.table`, `readxl` (preparação); `fixest`, `plm` (análise de painel); `officer`, `xml2` (saída). O setup deve verificar e instalar o que faltar.

---

## 8. Estrutura de pastas e entregáveis do projeto

```
<projeto>/
├── code/
│   ├── utils/
│   │   └── 00_setup.R          # paths, parâmetros (derivados!), helpers,
│   │                           # mapeamentos (nome→CNAE, campo_tec→CNAE, prefixos IPC)
│   ├── 01-prep/
│   │   ├── 01_read_pintec.R    # XLSX → pintec_cnae.rds
│   │   ├── 03_read_inpi_badepi.R # CSVs → badepi_cnae.rds
│   │   └── 05_build_pilares.R  # merge + (opcional) pilares/índice → painel_cnae_ano.rds
│   ├── 02-analysis/            # descritivas, índices, regressões (conforme o estudo)
│   └── 03-output/              # conversões (MD→DOCX), figuras, tables CSV
├── data/
│   ├── raw/                    # Original intocado (PINTEC por ano/, BADEPI)
│   ├── dictionaries/           # mapeamentos exportados (ex.: campo_tec_to_cnae.csv)
│   └── processed/              # pintec_cnae.rds, badepi_cnae.rds, painel_cnae_ano.rds
├── outputs/
│   ├── tables/                 # CSVs numerados (tab01_*.csv ...)
│   └── figures/                # PDFs numerados (fig01_*.pdf ...)
└── docs/                       # rascunhos, relatórios, verificação de referências
```

Convenções: scripts numerados por etapa; `saveRDS(compress = "xz")`; função `log_msg()` com timestamp prefixando TODA mensagem; `save_rds()`/`read_rds()` wrappers no setup.

---

## 9. Verificações de qualidade (imprima ao final de cada etapa)

**PINTEC:**
- [ ] Inventário de ondas/abas impresso antes da leitura
- [ ] nº de divisões CNAE por ano = esperado (22 nas ondas 2021–2024; reporte se divergir)
- [ ] nº de observações por onda (não pode faltar divisão inteira sem aviso)
- [ ] % de NAs por variável por ano (cobertura assimétrica documentada)
- [ ] Faixas das variáveis `pct_*` dentro de [0, 1] (ou razoável para contagens)
- [ ] Dedup `(cnae, ano)` executado; nº de duplicatas removidas registrado

**BADEPI:**
- [ ] Contagens após cada filtro (BR → IPC tema → CNAE mapeado → janela de anos)
- [ ] `range(ANO)` detectado e impresso
- [ ] RTA sem `Inf`; divisões sem patentes identificadas
- [ ] Cross-check opcional: total de pedidos BR vs. estatísticas publicadas INPI

**Painel:**
- [ ] Merge `all.x = TRUE`: nº de linhas do painel = nº de linhas PINTEC
- [ ] Painel ordenado por `(cnae, ano)`; sem duplicatas de chave
- [ ] Cobertura interseção/união de períodos entre fontes documentada

---

## 10. Protocolo de execução (ordem dos passos)

1. **Setup & inventário** — crie estrutura de pastas; liste arquivos brutos e abas; imprima inventário; derive períodos disponíveis.
2. **Parâmetros derivados** — preencha listas de anos/ondas a partir do inventário; apresente ao usuário para confirmação se algo divergir das referências deste documento.
3. **`01_read_pintec.R`** — extrair tabelas de interesse por onda → `pintec_cnae.rds`.
4. **`03_read_inpi_badepi.R`** — ler 4 CSVs essenciais, filtros, mapear CNAE, agregar → `badepi_cnae.rds`.
5. **`05_build_pilares.R`** — merge → `painel_cnae_ano.rds` (+ camada analítica se pedida).
6. **QA** — rodar todos os checks da seção 9; escrever relatório de auditoria (texto) em `docs/`.
7. **Entrega** — resumo executivo: períodos cobertos, nº de divisões, variáveis por fonte, limitações (seção 4.4), divergências encontradas e tratamento dado.

---

## 11. Formato do relatório final (obrigatório)

O relatório final ao usuário deve conter, no mínimo:

1. **Períodos detectados** por fonte (e a interseção usada no painel);
2. **Cobertura setoral** (divisões por onda; agregações tipo "05-09");
3. **Dicionário de variáveis** do painel (nome, fonte, aba/tabela de origem, fórmula, período de início);
4. **Contagens-chave** (obs por fonte; duplicatas removidas; linhas descartadas por filtro e motivo);
5. **Limitações** herdadas (BADEPI só-INPI, mapeamento aproximado, ausência de UF, cobertura assimétrica de temáticos);
6. **Divergências** entre estrutura real e este documento, e como foram resolvidas (ou por que ficaram pendentes de decisão do usuário).

---

## Checklist final de autoserviço (a IA deve confirmar antes de encerrar)

- [ ] Nenhum período foi hardcoded (grep por `202[0-9]` e `c(2021` no código — apenas em comentários/examples é aceitável)
- [ ] Nenhum arquivo em `data/raw/` foi modificado
- [ ] Todos os scripts rodam de ponta a ponta em sessão limpa (`Rscript` direto)
- [ ] Log de execução completo com timestamps
- [ ] Painel final salvo e com dicionário de variáveis documentado
- [ ] Relatório final no formato da seção 11 entregue
