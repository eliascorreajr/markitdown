# MarkItDown — Análise Completa do Projeto

> Versão do pacote analisada: **0.1.6b2**  
> Licença: **MIT**  
> Autor principal: Adam Fourney (Microsoft / AutoGen Team)  
> Repositório: https://github.com/microsoft/markitdown

---

## 1. O que é o MarkItDown?

**MarkItDown** é uma biblioteca Python leve para converter uma grande variedade de formatos de arquivo em **Markdown**. Seu foco é ser consumida por pipelines de análise de texto e modelos de linguagem grande (LLMs). O objetivo principal **não** é produzir um documento perfeitamente formatado para humanos, mas sim preservar a estrutura semântica do conteúdo (títulos, listas, tabelas, links) de forma que os LLMs possam processá-la com eficiência.

A ferramenta é comparável ao projeto `textract`, mas com ênfase em Markdown como formato de saída por sua alta compatibilidade com LLMs modernos (GPT-4o, Claude, Gemini, etc.) e eficiência em tokens.

---

## 2. Por que Markdown?

- Markdown é quase texto puro, com marcação mínima.
- LLMs foram treinados em grandes quantidades de texto formatado em Markdown e o "entendem" nativamente.
- Representa estrutura importante (títulos, listas, tabelas, links) sem sobrecarga.
- É altamente eficiente em número de tokens — importante para limites de contexto dos LLMs.

---

## 3. Estrutura do Repositório

```
markitdown/
├── Dockerfile                        # Imagem Docker com dependências (ffmpeg, exiftool)
├── .devcontainer/                    # Configuração de Dev Container (VS Code)
├── .github/                          # Workflows de CI e templates
├── .pre-commit-config.yaml           # Configurações de pré-commit
├── CODE_OF_CONDUCT.md                # Código de conduta (Microsoft Open Source)
├── LICENSE                           # Licença MIT
├── README.md                         # Documentação principal (inglês)
├── SECURITY.md                       # Política de segurança da Microsoft
├── SUPPORT.md                        # Canais de suporte
└── packages/
    ├── markitdown/                   # Pacote principal
    ├── markitdown-mcp/               # Servidor MCP (Model Context Protocol)
    ├── markitdown-ocr/               # Plugin OCR via LLM Vision
    └── markitdown-sample-plugin/     # Plugin de exemplo para desenvolvedores
```

---

## 4. Pacotes do Projeto

### 4.1 `markitdown` — Pacote Principal

É o núcleo do sistema. Contém a classe `MarkItDown`, todos os conversores embutidos, e a interface de linha de comando (CLI).

**Formatos suportados nativamente:**

| Formato | Dependência Opcional |
|---|---|
| PDF | `pdfminer.six`, `pdfplumber` |
| Word (.docx) | `mammoth`, `lxml` |
| PowerPoint (.pptx) | `python-pptx` |
| Excel (.xlsx) | `pandas`, `openpyxl` |
| Excel antigo (.xls) | `pandas`, `xlrd` |
| Imagens (EXIF + OCR) | `exiftool` (externo) |
| Áudio (transcrição) | `pydub`, `SpeechRecognition` |
| HTML | `beautifulsoup4`, `markdownify` |
| CSV | `pandas` |
| JSON / XML | `defusedxml` |
| ZIP | — (itera sobre conteúdo) |
| YouTube (legendas) | `youtube-transcript-api` |
| EPub | — |
| Outlook MSG | `olefile` |
| RSS / Atom | `beautifulsoup4` |
| Jupyter Notebook (.ipynb) | — |
| Bing SERP | — |
| Wikipedia | — |
| Azure Document Intelligence | `azure-ai-documentintelligence`, `azure-identity` |

---

### 4.2 `markitdown-mcp` — Servidor MCP

Expõe o MarkItDown como um servidor **MCP (Model Context Protocol)**, permitindo integração direta com aplicações de IA como **Claude Desktop**.

- Transportes suportados: **STDIO**, **Streamable HTTP**, **SSE**
- Expõe uma única ferramenta: `convert_to_markdown(uri)`
- URIs aceitas: `http:`, `https:`, `file:`, `data:`
- Por padrão, faz bind apenas em `localhost` (modo HTTP/SSE)

---

### 4.3 `markitdown-ocr` — Plugin OCR

Plugin que adiciona OCR via **LLM Vision** para PDFs, DOCX, PPTX e XLSX. Usa o mesmo par `llm_client` / `llm_model` já suportado pelo MarkItDown. Registra conversores com prioridade `-1.0` (acima dos conversores padrão).

---

### 4.4 `markitdown-sample-plugin` — Plugin de Exemplo

Template que demonstra como criar um plugin de terceiros. Ensina a implementar `DocumentConverter`, exportar `register_converters()` e declarar o entry point no `pyproject.toml`.

---

## 5. Instalação

### Pré-requisitos

- Python **3.10** ou superior
- Recomendado: ambiente virtual

```bash
python -m venv .venv
source .venv/bin/activate
```

### Instalação via pip

```bash
# Instalar tudo (backward-compatible)
pip install 'markitdown[all]'

# Instalar apenas formatos específicos
pip install 'markitdown[pdf,docx,pptx]'
```

### Instalação a partir do código-fonte

```bash
git clone https://github.com/microsoft/markitdown.git
cd markitdown
pip install -e 'packages/markitdown[all]'
```

### Dependências opcionais disponíveis

| Flag | O que instala |
|---|---|
| `[all]` | Todas as dependências opcionais |
| `[pptx]` | PowerPoint |
| `[docx]` | Word |
| `[xlsx]` | Excel moderno |
| `[xls]` | Excel antigo |
| `[pdf]` | PDF |
| `[outlook]` | Outlook MSG |
| `[az-doc-intel]` | Azure Document Intelligence |
| `[audio-transcription]` | Transcrição de áudio (WAV/MP3) |
| `[youtube-transcription]` | Legendas do YouTube |

---

## 6. Como Usar

### 6.1 Linha de Comando (CLI)

```bash
# Converter um arquivo e imprimir na saída padrão
markitdown arquivo.pdf

# Salvar em arquivo
markitdown arquivo.pdf -o saida.md

# Ler da entrada padrão (stdin)
cat arquivo.pdf | markitdown

# Converter com Azure Document Intelligence
markitdown arquivo.pdf -d -e "https://seu-endpoint.cognitiveservices.azure.com/"

# Usar plugins de terceiros
markitdown --use-plugins arquivo.rtf

# Listar plugins instalados
markitdown --list-plugins

# Passar dica de extensão quando lendo de stdin
cat arquivo | markitdown -x pdf

# Passar dica de MIME type
markitdown arquivo -m application/pdf

# Manter data URIs (base64) na saída
markitdown arquivo.html --keep-data-uris
```

### 6.2 API Python

```python
from markitdown import MarkItDown

# Uso básico
md = MarkItDown()
resultado = md.convert("documento.pdf")
print(resultado.markdown)

# Com Azure Document Intelligence
md = MarkItDown(docintel_endpoint="https://seu-endpoint.cognitiveservices.azure.com/")
resultado = md.convert("documento.pdf")

# Com LLM para descrição de imagens
from openai import OpenAI
client = OpenAI()
md = MarkItDown(llm_client=client, llm_model="gpt-4o")
resultado = md.convert("imagem.jpg")

# Com plugins habilitados
md = MarkItDown(enable_plugins=True)
resultado = md.convert("arquivo.rtf")

# Converter stream binário
import io
with open("arquivo.pdf", "rb") as f:
    resultado = md.convert_stream(f)

# Converter URL
resultado = md.convert("https://exemplo.com/pagina.html")

# Converter data URI
resultado = md.convert("data:text/plain;base64,SGVsbG8gV29ybGQ=")
```

### 6.3 Via Docker

```bash
# Build da imagem
docker build -t markitdown:latest .

# Converter arquivo via pipe
docker run --rm -i markitdown:latest < arquivo.pdf > saida.md
```

### 6.4 Servidor MCP (Claude Desktop)

```bash
# Instalar
pip install markitdown-mcp

# Iniciar servidor STDIO
markitdown-mcp

# Iniciar servidor HTTP
markitdown-mcp --http --host 127.0.0.1 --port 3001
```

Configuração no `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "markitdown": {
      "command": "docker",
      "args": ["run", "--rm", "-i", "markitdown-mcp:latest"]
    }
  }
}
```

---

## 7. Sistema de Plugins

### Como funciona

1. Plugins são pacotes Python normais instalados no mesmo ambiente.
2. Plugins são descobertos via **entry points** do Python (`markitdown.plugin` group).
3. Por padrão, plugins estão **desabilitados**. Devem ser ativados explicitamente com `enable_plugins=True` ou `--use-plugins`.
4. O método `register_converters(markitdown, **kwargs)` do plugin é chamado durante a construção do `MarkItDown`.

### Criar um plugin

```python
from markitdown import MarkItDown, DocumentConverter, DocumentConverterResult, StreamInfo
from typing import BinaryIO, Any

class MeuConverter(DocumentConverter):
    def accepts(self, file_stream: BinaryIO, stream_info: StreamInfo, **kwargs: Any) -> bool:
        return stream_info.extension == ".meu"

    def convert(self, file_stream: BinaryIO, stream_info: StreamInfo, **kwargs: Any) -> DocumentConverterResult:
        conteudo = file_stream.read().decode("utf-8")
        return DocumentConverterResult(markdown=conteudo)

__plugin_interface_version__ = 1

def register_converters(markitdown: MarkItDown, **kwargs):
    markitdown.register_converter(MeuConverter())
```

No `pyproject.toml`:

```toml
[project.entry-points."markitdown.plugin"]
meu_plugin = "meu_pacote"
```

---

## 8. Plugin OCR em Prática

```bash
pip install markitdown-ocr openai
```

```python
from markitdown import MarkItDown
from openai import OpenAI

md = MarkItDown(
    enable_plugins=True,
    llm_client=OpenAI(),
    llm_model="gpt-4o",
)
resultado = md.convert("documento_com_imagens.pdf")
print(resultado.markdown)
```

Cada bloco OCR é formatado como:
```
*[Image OCR]
<texto extraído>
[End OCR]*
```

---

## 9. Testes e Desenvolvimento

```bash
cd packages/markitdown

# Instalar o hatch
pip install hatch

# Abrir shell do ambiente de teste
hatch shell

# Rodar testes
hatch test

# Checar pré-commit antes do PR
pre-commit run --all-files

# Testes do plugin OCR
cd packages/markitdown-ocr
pytest tests/ -v
```

---

## 10. Segurança

### 10.1 Política Geral (Microsoft MSRC)

- Vulnerabilidades devem ser reportadas **privadamente** ao Microsoft Security Response Center (MSRC): https://msrc.microsoft.com/create-report
- **Não** abrir issues públicas no GitHub para reportar vulnerabilidades.
- Email alternativo: secure@microsoft.com (suporta PGP)
- Prazo de resposta: até 24 horas.
- Microsoft segue o princípio de **Coordinated Vulnerability Disclosure (CVD)**.

### 10.2 Segurança no Servidor MCP

- O servidor MCP **não tem autenticação** e executa com os privilégios do usuário.
- Em modo HTTP/SSE, faz bind **somente em localhost** por padrão.
- A ferramenta `convert_to_markdown` pode ler **qualquer arquivo** ao qual o usuário tenha acesso.
- **Nunca** expor o servidor em interfaces não-localhost sem entender as implicações.
- Recomendado: executar em container ou VM com permissões mínimas.

### 10.3 Parsing Seguro de XML

- O projeto usa `defusedxml` para parsear XML/HTML, prevenindo ataques de **XXE (XML External Entity)** e **Billion Laughs**.

### 10.4 Imagens Docker

- A imagem Docker executa como usuário `nobody:nogroup` por padrão (sem root).
- Variável `INSTALL_GIT` permite controlar se `git` é instalado (padrão: `false`).

### 10.5 Processamento de Arquivos Externos

- Ao converter URLs, o MarkItDown faz requisições HTTP usando `requests.Session`.
- Arquivos de entrada não são sanitizados de conteúdo malicioso além das bibliotecas usadas (pdfplumber, mammoth, etc.).
- Recomenda-se não expor o serviço a entradas não confiáveis sem sandbox adicional.

### 10.6 Dependências de Terceiros

- Todas as dependências opcionais possuem versões fixadas ou intervalos específicos em `pyproject.toml`, reduzindo riscos de supply chain.
- Notificações de terceiros estão documentadas em `packages/markitdown/ThirdPartyNotices.md`.

---

## 11. Contribuição

- O projeto aceita pull requests e issues.
- Contribuidores precisam assinar o **CLA (Contributor License Agreement)** da Microsoft.
- Issues marcadas com `open for contribution` são boas entradas para novos contribuidores.
- Siga o [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).

---

## 12. Referências

- Repositório: https://github.com/microsoft/markitdown
- PyPI (markitdown): https://pypi.org/project/markitdown/
- PyPI (markitdown-mcp): https://pypi.org/project/markitdown-mcp/
- Azure Document Intelligence: https://learn.microsoft.com/azure/ai-services/document-intelligence/
- Model Context Protocol: https://modelcontextprotocol.io/
- Buscar plugins: `#markitdown-plugin` no GitHub
