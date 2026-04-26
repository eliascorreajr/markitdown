# MarkItDown — Descrição Técnica (Schematics)

> Versão: **0.1.6b2** | Licença: MIT | Python ≥ 3.10

---

## 1. Arquitetura Geral

O MarkItDown é organizado em um **monorepo** com quatro pacotes independentes:

```
packages/
├── markitdown/             # Núcleo: conversores, classe MarkItDown, CLI
├── markitdown-mcp/         # Servidor MCP (STDIO / HTTP / SSE)
├── markitdown-ocr/         # Plugin OCR via LLM Vision
└── markitdown-sample-plugin/  # Template para plugins de terceiros
```

Cada pacote é um projeto Python independente com seu próprio `pyproject.toml` e pode ser publicado separadamente no PyPI.

---

## 2. Módulos do Pacote `markitdown`

```
src/markitdown/
├── __about__.py            # Versão do pacote (__version__)
├── __init__.py             # Exportações públicas
├── __main__.py             # Entry point da CLI
├── _base_converter.py      # Classes base: DocumentConverter, DocumentConverterResult
├── _exceptions.py          # Hierarquia de exceções
├── _markitdown.py          # Classe principal MarkItDown + lógica de conversão
├── _stream_info.py         # Dataclass StreamInfo (metadados de stream)
├── _uri_utils.py           # Utilitários para parsing de URIs (file:, data:)
├── converter_utils/        # Utilitários compartilhados pelos conversores
└── converters/             # Um arquivo por conversor embutido
    ├── _plain_text_converter.py
    ├── _html_converter.py
    ├── _rss_converter.py
    ├── _wikipedia_converter.py
    ├── _youtube_converter.py
    ├── _ipynb_converter.py
    ├── _bing_serp_converter.py
    ├── _pdf_converter.py
    ├── _docx_converter.py
    ├── _xlsx_converter.py       # XlsxConverter + XlsConverter
    ├── _pptx_converter.py
    ├── _image_converter.py
    ├── _audio_converter.py
    ├── _outlook_msg_converter.py
    ├── _zip_converter.py
    ├── _doc_intel_converter.py  # DocumentIntelligenceConverter
    ├── _epub_converter.py
    ├── _csv_converter.py
    ├── _exiftool.py             # Wrapper para ExifTool externo
    ├── _llm_caption.py          # Geração de descrição via LLM
    ├── _markdownify.py          # Customizações do markdownify
    └── _transcribe_audio.py     # Wrapper para SpeechRecognition
```

---

## 3. Classes Principais

### 3.1 `DocumentConverter` (abstrata)

```
_base_converter.py → DocumentConverter
```

Interface que todos os conversores devem implementar:

| Método | Assinatura | Descrição |
|---|---|---|
| `accepts()` | `(file_stream, stream_info, **kwargs) → bool` | Decide se o conversor pode lidar com o arquivo |
| `convert()` | `(file_stream, stream_info, **kwargs) → DocumentConverterResult` | Realiza a conversão |

- `file_stream`: `BinaryIO` — o arquivo como stream de bytes (deve suportar `seek`, `tell`, `read`)
- `stream_info`: `StreamInfo` — metadados sobre o arquivo
- Após `accepts()` ler dados do stream, **deve resetar a posição** com `file_stream.seek(cur_pos)`

### 3.2 `DocumentConverterResult`

```
_base_converter.py → DocumentConverterResult
```

| Atributo | Tipo | Descrição |
|---|---|---|
| `markdown` | `str` | Conteúdo convertido em Markdown |
| `title` | `Optional[str]` | Título do documento (quando disponível) |
| `text_content` | `str` (alias) | Apelido depreciado de `markdown` |

Implementa `__str__()` retornando `self.markdown`.

### 3.3 `StreamInfo` (dataclass frozen)

```
_stream_info.py → StreamInfo
```

| Campo | Tipo | Origem |
|---|---|---|
| `mimetype` | `Optional[str]` | Cabeçalho HTTP, detecção Magika, ou hint |
| `extension` | `Optional[str]` | Extensão do arquivo |
| `charset` | `Optional[str]` | Cabeçalho HTTP ou hint |
| `filename` | `Optional[str]` | Nome do arquivo ou Content-Disposition |
| `local_path` | `Optional[str]` | Caminho no disco |
| `url` | `Optional[str]` | URL de origem |

Método `copy_and_update(*args, **kwargs)` cria uma cópia com campos atualizados (campos `None` não sobrescrevem).

### 3.4 `MarkItDown` (classe principal)

```
_markitdown.py → MarkItDown
```

**Construtor:**

```python
MarkItDown(
    enable_builtins=None,     # True por padrão
    enable_plugins=None,      # False por padrão
    llm_client=None,          # Cliente OpenAI-compatible para imagens
    llm_model=None,           # Modelo LLM (e.g. "gpt-4o")
    llm_prompt=None,          # Prompt customizado para descrição de imagens
    exiftool_path=None,       # Caminho para o executável exiftool
    style_map=None,           # Mapeamento de estilos para mammoth (DOCX)
    docintel_endpoint=None,   # Endpoint do Azure Document Intelligence
    docintel_credential=None, # Credencial Azure (DefaultAzureCredential por padrão)
    docintel_file_types=None, # Tipos de arquivo para o DocIntel
    docintel_api_version=None,# Versão da API do DocIntel
    requests_session=None,    # requests.Session customizada
)
```

**Métodos de conversão públicos:**

| Método | Aceita |
|---|---|
| `convert(source)` | `str` (caminho ou URL), `Path`, `requests.Response`, `BinaryIO` |
| `convert_local(path)` | `str` ou `Path` |
| `convert_stream(stream)` | `BinaryIO` |
| `convert_uri(uri)` | `file:`, `data:`, `http:`, `https:` |
| `convert_url(url)` | alias de `convert_uri` (legado) |
| `convert_response(response)` | `requests.Response` |

**Métodos de gerenciamento de conversores:**

| Método | Descrição |
|---|---|
| `register_converter(converter, priority)` | Registra um conversor com prioridade |
| `enable_builtins(**kwargs)` | Registra conversores embutidos (chamado no `__init__`) |
| `enable_plugins(**kwargs)` | Descobre e registra plugins via entry points |

### 3.5 `ConverterRegistration` (dataclass frozen)

```python
@dataclass(kw_only=True, frozen=True)
class ConverterRegistration:
    converter: DocumentConverter
    priority: float
```

Usado internamente para ordenar os conversores por prioridade.

---

## 4. Hierarquia de Exceções

```
MarkItDownException (base)
├── MissingDependencyException   # Dependência opcional não instalada
├── UnsupportedFormatException   # Nenhum conversor encontrado para o formato
└── FileConversionException      # Conversão falhou (lista de tentativas disponível)

FailedConversionAttempt          # Não é exceção; armazena converter + exc_info por tentativa
```

---

## 5. Fluxo de Conversão

```
convert(source)
    │
    ├─ str (caminho local) ──► convert_local()
    ├─ str (URL/URI)        ──► convert_uri()
    ├─ Path                 ──► convert_local()
    ├─ requests.Response    ──► convert_response()
    └─ BinaryIO             ──► convert_stream()
                                    │
                        _get_stream_info_guesses()
                        (Magika detecta MIME + combina hints)
                                    │
                             _convert(file_stream, guesses)
                                    │
                    Para cada StreamInfo guess (por prioridade):
                        Para cada conversor registrado (por prioridade):
                            if converter.accepts(stream, guess):
                                resultado = converter.convert(stream, guess)
                                return resultado
                                    │
                    Se nenhum conversor aceitar: UnsupportedFormatException
                    Se todos falharem: FileConversionException (com tentativas)
```

### Detecção de tipo de arquivo

O MarkItDown usa **dois mecanismos** em cascata para detectar o tipo de arquivo:

1. **Informação fornecida** (`StreamInfo` manual ou cabeçalhos HTTP)
2. **Magika** (ML-based file type detector do Google) — detecta por conteúdo do stream

Os "guesses" resultantes são testados em sequência pelo `_convert()`.

### Prioridade dos conversores

| Constante | Valor | Uso |
|---|---|---|
| `PRIORITY_SPECIFIC_FILE_FORMAT` | `0.0` | Conversores específicos (PDF, DOCX, etc.) |
| `PRIORITY_GENERIC_FILE_FORMAT` | `10.0` | Conversores genéricos (texto, HTML) |
| Plugins OCR | `-1.0` | Antes dos embutidos |

**Menor valor = maior prioridade** (tentado primeiro). Conversores registrados por último dentro da mesma prioridade são tentados primeiro (LIFO por prioridade).

---

## 6. Sistema de Plugins

### Descoberta

```python
from importlib.metadata import entry_points
entry_points(group="markitdown.plugin")
```

Os plugins são carregados **uma vez** (lazy, cacheado em `_plugins` global). Falhas de carregamento geram apenas um `warning` e são ignoradas.

### Interface obrigatória do plugin

```python
# Versão da interface (somente 1 suportada atualmente)
__plugin_interface_version__ = 1

# Entry point chamado pelo MarkItDown
def register_converters(markitdown: MarkItDown, **kwargs):
    markitdown.register_converter(MeuConverter())
```

### Entry point no `pyproject.toml`

```toml
[project.entry-points."markitdown.plugin"]
nome_do_plugin = "nome_do_modulo_python"
```

---

## 7. Conversores Embutidos — Detalhes Técnicos

| Conversor | MIME / Extensão | Dependências | Observações |
|---|---|---|---|
| `PlainTextConverter` | `text/*` | — | Catch-all para texto |
| `HtmlConverter` | `text/html` | `beautifulsoup4`, `markdownify` | Usa markdownify customizado |
| `RssConverter` | RSS/Atom feeds | `beautifulsoup4` | |
| `WikipediaConverter` | URLs wikipedia.org | `beautifulsoup4` | Detecta via URL |
| `YouTubeConverter` | URLs youtube.com | `youtube-transcript-api` | Busca transcrição via API |
| `BingSerpConverter` | URLs bing.com/search | `beautifulsoup4` | Extrai resultados SERP |
| `IpynbConverter` | `.ipynb` | — | Parseia JSON do notebook |
| `PdfConverter` | `application/pdf` | `pdfminer.six`, `pdfplumber` | |
| `DocxConverter` | `.docx` | `mammoth`, `lxml` | HTML intermediário |
| `XlsxConverter` | `.xlsx` | `pandas`, `openpyxl` | DataFrame → tabela Markdown |
| `XlsConverter` | `.xls` | `pandas`, `xlrd` | |
| `PptxConverter` | `.pptx` | `python-pptx` | Usa LLM para imagens se configurado |
| `ImageConverter` | `image/*` | `exiftool` (externo) | EXIF + LLM caption opcional |
| `AudioConverter` | `audio/*` | `pydub`, `SpeechRecognition`, `ffmpeg` | Transcrição + EXIF |
| `OutlookMsgConverter` | `.msg` | `olefile` | Lê magic bytes para aceitar |
| `ZipConverter` | `application/zip` | — | Itera recursivamente via `MarkItDown` |
| `EpubConverter` | `.epub` | — | ZIP especial com HTML interno |
| `CsvConverter` | `text/csv` | `pandas` | DataFrame → tabela Markdown |
| `DocumentIntelligenceConverter` | múltiplos | `azure-ai-documentintelligence` | Registrado somente se endpoint configurado |

---

## 8. Módulo MCP (`markitdown-mcp`)

### Arquitetura

- Usa o SDK `mcp` (Model Context Protocol).
- Transportes:
  - **STDIO**: padrão; processo filho do LLM client (Claude Desktop, etc.)
  - **Streamable HTTP** (`/mcp`): HTTP moderno com suporte a streaming
  - **SSE** (`/sse`): Server-Sent Events para compatibilidade legada

### Ferramenta exposta

```
convert_to_markdown(uri: str) -> str
```

- Internamente instancia `MarkItDown` e chama `convert(uri)`.
- Aceita `http:`, `https:`, `file:`, `data:` URIs.

### Configuração CLI

```
markitdown-mcp [--http] [--host HOST] [--port PORT]
```

### Segurança

- Sem autenticação.
- Bind em `127.0.0.1` por padrão no modo HTTP/SSE.
- Pode acessar qualquer arquivo no sistema com as permissões do processo.

---

## 9. Plugin OCR — Arquitetura

### Fluxo de registro

```
MarkItDown(enable_plugins=True, llm_client=..., llm_model=...)
    │
    └─ enable_plugins(**kwargs)
            │
            └─ plugin.register_converters(markitdown, llm_client=..., llm_model=...)
                    │
                    └─ Cria LLMVisionOCRService(llm_client, llm_model, llm_prompt)
                    └─ Registra:
                        - OCRPdfConverter      (priority=-1.0)
                        - OCRDocxConverter     (priority=-1.0)
                        - OCRPptxConverter     (priority=-1.0)
                        - OCRXlsxConverter     (priority=-1.0)
```

### Formato de saída OCR

```
*[Image OCR]
<texto extraído pela LLM Vision>
[End OCR]*
```

### Comportamento por formato

| Formato | Estratégia |
|---|---|
| PDF | Extrai imagens por XObject; fallback página inteira a 300 DPI para PDFs escaneados; fallback PyMuPDF para PDFs malformados |
| DOCX | Extrai via `doc.part.rels`; injeta placeholders antes do pipeline HTML→MD |
| PPTX | Processa shapes em ordem top-left; LLM description first, OCR como fallback |
| XLSX | Extrai de `sheet._images`; lista sob `### Images in this sheet:` |

---

## 10. Detecção de Tipo de Arquivo com Magika

O MarkItDown usa [Magika](https://github.com/google/magika) (Google) para detecção de tipo por **conteúdo** (ML-based), em vez de depender apenas de extensão ou MIME type declarado.

```python
self._magika = magika.Magika()
# Usado em _get_stream_info_guesses()
```

A detecção por conteúdo é mais robusta contra:
- Arquivos com extensão errada
- Streams sem extensão (stdin)
- Arquivos com Content-Type incorreto

---

## 11. Parsing Seguro (Segurança Técnica)

### `defusedxml`

Toda leitura de XML/HTML usa `defusedxml` ao invés de `xml.etree.ElementTree` padrão:

- Previne **XXE (XML External Entity Injection)**
- Previne **Billion Laughs (XML bomb)**
- Previne **Quadratic blowup**

### Streams binários imutáveis

- `StreamInfo` é um **dataclass frozen** — imutável após criação.
- Alterações são feitas através de `copy_and_update()`, que cria um novo objeto.

### Execução de processos externos (ExifTool, FFmpeg)

- `exiftool_path` é validado contra uma lista de diretórios confiáveis conhecidos.
- `EXIFTOOL_PATH` e `FFMPEG_PATH` podem ser configurados via variáveis de ambiente.
- A imagem Docker define esses caminhos explicitamente.

### Requisições HTTP

- `requests.Session` com header `Accept: text/markdown, text/html;q=0.9, ...`
- Sem timeout padrão configurado (a ser observado em integrações).

---

## 12. Sistema de Build e Testes

### Ferramenta de build: `hatch`

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

Ambientes definidos:
- `default`: `features = ["all"]`
- `hatch-test`: `features = ["all"]` + `openai` extra
- `types`: `features = ["all"]` + `mypy`

### Executar testes

```bash
cd packages/markitdown
hatch test
```

### Type checking

```bash
hatch run types:check
```

### Coverage

Configurado em `pyproject.toml`:
- `branch = true` (cobertura de ramificações)
- `parallel = true`
- Exclui `__about__.py` e `__main__`

### Pre-commit hooks

```bash
pre-commit run --all-files
```

---

## 13. Diagrama de Dependências (Pacote Principal)

```
markitdown (core)
├── beautifulsoup4      # HTML parsing
├── requests            # HTTP
├── markdownify         # HTML → Markdown
├── magika~=0.6.1       # Detecção de tipo por ML
├── charset-normalizer  # Detecção de charset
└── defusedxml          # XML seguro

Opcionais:
├── python-pptx         → PptxConverter
├── mammoth~=1.11.0     → DocxConverter
├── lxml                → DocxConverter
├── pandas              → XlsxConverter, XlsConverter, CsvConverter
├── openpyxl            → XlsxConverter
├── xlrd                → XlsConverter
├── pdfminer.six        → PdfConverter
├── pdfplumber          → PdfConverter
├── olefile             → OutlookMsgConverter
├── pydub               → AudioConverter
├── SpeechRecognition   → AudioConverter
├── youtube-transcript-api → YouTubeConverter
├── azure-ai-documentintelligence → DocumentIntelligenceConverter
└── azure-identity      → DocumentIntelligenceConverter
```

---

## 14. Variáveis de Ambiente

| Variável | Uso |
|---|---|
| `EXIFTOOL_PATH` | Caminho completo para o executável `exiftool` |
| `FFMPEG_PATH` | Caminho completo para o executável `ffmpeg` |

Definidos na imagem Docker em `/usr/bin/exiftool` e `/usr/bin/ffmpeg`.

---

## 15. Exportações Públicas (`__init__.py`)

```python
from markitdown import (
    MarkItDown,
    DocumentConverter,
    DocumentConverterResult,
    StreamInfo,
    # Exceções
    MarkItDownException,
    MissingDependencyException,
    UnsupportedFormatException,
    FileConversionException,
    FailedConversionAttempt,
    # Conversores (para uso em plugins)
    PlainTextConverter,
    HtmlConverter,
    PdfConverter,
    DocxConverter,
    XlsxConverter,
    XlsConverter,
    PptxConverter,
    ImageConverter,
    AudioConverter,
    # ... todos os outros conversores
)
```
