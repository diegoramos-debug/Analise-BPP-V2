# 🚀 Deploy Automático — GitHub Pages

> Instrução complementar ao `GUIA_PROJETO_RELATORIO_PERDAS_v2_4.md`
> Atualizado em: 26/06/2026

---

## Repositório vinculado

| Item | Valor |
|---|---|
| **Repositório** | `diegoramos-debug/Analise-BPP-V2` |
| **Branch** | `main` |
| **Site público (GitHub Pages)** | https://diegoramos-debug.github.io/Analise-BPP-V2/ |
| **Arquivo principal do site** | `index.html` (raiz do repo) |
| **Arquivo de backup versionado** | `relatorio_perdas.html` (raiz do repo) |

> O GitHub Pages está configurado para servir a partir da raiz (`/`) do branch `main`.  
> Portanto, **`index.html` é o que aparece no link público**. Sempre atualize os dois arquivos.

---

## Como funciona o deploy

Ao final de cada geração de relatório, o script faz upload via **GitHub API (PUT /contents)**:

1. Busca o SHA atual do arquivo no repo (necessário para sobrescrever)
2. Codifica o HTML em Base64
3. Envia o novo conteúdo com mensagem de commit automática
4. O GitHub Pages republica em ~30 segundos

---

## Script de deploy (embutir no final do `build_final.py`)

Adicionar este bloco **após** o `with open('relatorio_perdas.html'...)`:

```python
# ── Deploy automático GitHub Pages ────────────────────────────────────────────
import base64, urllib.request, urllib.error

GITHUB_TOKEN = "<SEU_GITHUB_TOKEN>"
GITHUB_REPO  = "diegoramos-debug/Analise-BPP-V2"
GITHUB_BRANCH = "main"

def github_put(path_in_repo, local_file, commit_msg):
    """Faz upload/atualização de um arquivo no GitHub via API."""
    api_url = f"https://api.github.com/repos/{GITHUB_REPO}/contents/{path_in_repo}"
    headers = {
        "Authorization": f"token {GITHUB_TOKEN}",
        "Accept": "application/vnd.github.v3+json",
        "Content-Type": "application/json"
    }
    # Ler arquivo local e codificar
    with open(local_file, "rb") as f:
        content_b64 = base64.b64encode(f.read()).decode()

    # Buscar SHA atual (necessário para atualizar arquivo existente)
    sha = None
    req = urllib.request.Request(api_url, headers=headers)
    try:
        with urllib.request.urlopen(req) as r:
            sha = json.loads(r.read()).get("sha")
    except urllib.error.HTTPError:
        pass  # Arquivo ainda não existe — será criado

    payload = {"message": commit_msg, "content": content_b64, "branch": GITHUB_BRANCH}
    if sha:
        payload["sha"] = sha

    req2 = urllib.request.Request(api_url, data=json.dumps(payload).encode(), headers=headers, method="PUT")
    with urllib.request.urlopen(req2) as r:
        return json.loads(r.read())["content"]["html_url"]

svc_label = D.get("svc", "").replace(" | ", "-")
commit_msg = f"deploy: relatorio {svc_label} W{D['week_min'].split('-')[1]}-W{D['week_max'].split('-')[1]}/{D['week_max'].split('-')[0]}"

print("\n🚀 Fazendo deploy no GitHub Pages...")
try:
    url1 = github_put("index.html",            "relatorio_perdas.html", commit_msg)
    url2 = github_put("relatorio_perdas.html", "relatorio_perdas.html", commit_msg)
    print(f"   ✅ index.html     → {url1}")
    print(f"   ✅ relatorio.html → {url2}")
    print(f"   🌐 Site público   → https://diegoramos-debug.github.io/Analise-BPP-V2/")
except Exception as e:
    print(f"   ⚠️  Erro no deploy: {e}")
```

---

## Execução completa com deploy

```bash
# 1. Atualizar CSV_PATH em build_data.py
# 2. Gerar dados
python3 build_data.py

# 3. Gerar HTML + fazer deploy automático
python3 build_final.py
```

**Saída esperada ao final:**

```
Total size: 461493

🚀 Fazendo deploy no GitHub Pages...
   ✅ index.html     → https://github.com/diegoramos-debug/Analise-BPP-V2/blob/main/index.html
   ✅ relatorio.html → https://github.com/diegoramos-debug/Analise-BPP-V2/blob/main/relatorio_perdas.html
   🌐 Site público   → https://diegoramos-debug.github.io/Analise-BPP-V2/
```

---

## Troubleshooting de deploy

| Sintoma | Causa | Fix |
|---|---|---|
| `HTTP Error 401` | Token inválido ou expirado | Gerar novo token em GitHub → Settings → Developer Settings → Personal Access Tokens (escopos: `repo`, `workflow`) |
| `HTTP Error 404` | Repositório não encontrado ou privado sem acesso | Confirmar `GITHUB_REPO` e permissões do token |
| `HTTP Error 422` | SHA incorreto (conflito de versão) | Rodar novamente — o script sempre busca o SHA atual antes de enviar |
| Site não atualiza | GitHub Pages ainda buildando | Aguardar ~60s e recarregar. Se persistir, checar aba *Actions* no repo |
| `index.html` atualiza mas site mostra versão antiga | Cache do browser | Forçar `Ctrl+Shift+R` ou abrir em aba anônima |

---

## Estrutura final do repositório

```
Analise-BPP-V2/
├── index.html              ← relatório mais recente (serve o GitHub Pages)
└── relatorio_perdas.html   ← mesmo arquivo, mantido como backup nomeado
```

---

*Projeto: Análise de Perdas Logísticas BPP USD / LM MELI — Deploy v1.0*
