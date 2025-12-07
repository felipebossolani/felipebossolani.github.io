---
layout: post
title: "Como usar o Claude Code com modelos do OpenRouter (incluindo free tiers) usando o Claude Code Router"
date: 2025-12-07 12:30:00 -0300
categories: [Claude Code, OpenRouter, Tutorial]
tags: [ai, coding, llm, open source]
---

Este é um guia completo para configurar o [Claude Code](https://docs.anthropic.com/en/docs/agents-and-tools/claude-code/overview){:target="_blank"} para funcionar com qualquer modelo do [OpenRouter](https://openrouter.ai){:target="_blank"} (incluindo modelos gratuitos) através do [Claude Code Router (CCR)](https://github.com/musistudio/claude-code-router){:target="_blank"}.

## 1. Introdução

O **Claude Code** é hoje um dos melhores agentes de programação interativos. No entanto, por padrão, ele é limitado aos modelos da Anthropic (Sonnet, Opus, Haiku).

O **Claude Code Router (CCR)** — um projeto open source — resolve isso permitindo:
*   Usar qualquer LLM compatível com a API da OpenAI (incluindo OpenRouter).
*   Trocar de modelo facilmente via comando `/model provider,model`.
*   Aproveitar o ambiente completo do Claude Code sem ficar preso a um único provedor.

Neste tutorial, você aprenderá passo a passo como ativar o Claude Code + OpenRouter, incluindo onde obter sua API key, como ajustar o `config.json` e como trocar modelos dentro do Claude Code.

> **Nota:** Screenshot da interface do Claude Code antes da configuração (menu de modelos Anthropic) seria inserida aqui.

---

## 2. Pré-requisitos

Antes de começar, certifique-se de ter:

*   macOS ou Linux com Homebrew instalado.
*   Node.js 18+.
*   Claude CLI instalado (via Brew).
*   Acesso ao [OpenRouter](https://openrouter.ai){:target="_blank"}.
*   API key válida para "Programmatic Access".

**Comandos úteis de verificação:**

```bash
claude --version
node -v
brew --version
```

---

## 3. Instalar o Claude Code Router (CCR)

O CCR pode ser instalado facilmente via Homebrew:

```bash
brew install claude-code-router
```

Verifique se o binário está disponível e instalado corretamente:

```bash
which ccr
ccr --help
```

> **Nota:** Screenshot do terminal mostrando `ccr --version` seria inserida aqui.

---

## 4. Obter a API Key no OpenRouter

1.  Acesse: [https://openrouter.ai/settings/keys](https://openrouter.ai/settings/keys)
2.  Clique em **"Create API Key"**.
3.  Escolha o tipo **"Programmatic API Key"**.
4.  Copie a key gerada (`sk-or-...`).
5.  **Salve em um lugar seguro** — ela não poderá ser visualizada novamente.

> **Nota:** Screenshot da página de criação de API Keys no OpenRouter seria inserida aqui.

---

## 5. Criar o arquivo de configuração do CCR

O CCR lê suas configurações do arquivo `~/.claude-code-router/config.json`.

Crie a pasta caso ela não exista e abra o arquivo para edição:

```bash
mkdir -p ~/.claude-code-router
nano ~/.claude-code-router/config.json
```

Abaixo, um exemplo de configuração minimalista usando OpenRouter e o modelo **Kat Coder Pro Free**:

```json
{
  "LOG": true,
  "API_TIMEOUT_MS": 600000,
  "Providers": [
    {
      "name": "openrouter",
      "api_base_url": "https://openrouter.ai/api/v1/chat/completions",
      "api_key": "${OPENROUTER_API_KEY}",
      "models": [
        "kwaipilot/kat-coder-pro:free"
      ],
      "transformer": {
        "use": ["openrouter"]
      }
    }
  ],
  "Router": {
    "default": "openrouter,kwaipilot/kat-coder-pro:free",
    "longContextThreshold": 60000
  }
}
```
![Configuração do CCR no terminal](/assets/images/ccr-config-terminal.png)
*Arquivo de configuração do CCR aberto no editor.*

---

## 6. Exportar a variável OPENROUTER_API_KEY

Para segurança, não colocamos a chave diretamente no JSON. Exporte-a em seu `~/.zshrc` ou `~/.bashrc`:

```bash
export OPENROUTER_API_KEY="sk-or-sua-chave-aqui"
```

Carregue as alterações no terminal atual:

```bash
source ~/.zshrc
echo $OPENROUTER_API_KEY
```

---

## 7. Reiniciar o CCR

Reinicie o CCR para que ele leia a nova variável de ambiente e configurações:

```bash
ccr restart
```

Teste se o modelo está acessível diretamente pelo CCR:

```bash
ccr model run openrouter,kwaipilot/kat-coder-pro:free "hello"
```

Se receber uma resposta, tudo está configurado corretamente!

> **Nota:** Capture da resposta "hello" retornando do modelo via CCR seria inserida aqui.

---

## 8. Usar o CCR com o Claude Code

Existem duas formas de integrar o CCR ao Claude Code.

### 8.1. Modo recomendado: Ativar o shell (Backend do Claude)

Este método redireciona o Claude Code para usar o CCR como backend.

Rode:

```bash
ccr start
eval "$(ccr activate)"
```

Isso ajusta variáveis de ambiente como `ANTHROPIC_BASE_URL=http://127.0.0.1:3456`.

Agora, use o Claude Code naturalmente:

```bash
claude code
```

O backend agora será o OpenRouter com seus modelos configurados.

### 8.2. Modo alternativo: Usar diretamente `ccr code`

Você também pode rodar diretamente:

```bash
ccr code
```

Esse método ignora o backend da Anthropic completamente.

---

## 9. Como trocar de modelo dentro do Claude Code via CCR

O menu `/model` do Claude Code continuará mostrando apenas Sonnet/Opus/Haiku. Para usar os modelos do OpenRouter, você deve usar o comando manual:

```text
/model openrouter,kwaipilot/kat-coder-pro:free
```

Outros exemplos de uso:

```text
/model openrouter,anthropic/claude-3.5-sonnet
/model openrouter,deepseek/deepseek-reasoner
/model openrouter,google/gemini-2.5-pro
```

> **Nota:** Screenshot da linha de comando do Claude Code mostrando o comando `/model` seria inserida aqui.

```
![Seleção de modelo no Claude Code](/assets/images/claude-code-model-select.png)
*Menu de seleção de modelos mostrando o modelo customizado do OpenRouter.*

---

## 10. Logs & Debug

Para verificar o que está acontecendo "por baixo dos panos":

```bash
ls ~/.claude-code-router/logs
tail -f ~/.claude-code-router/logs/ccr-*.log
```

Isso mostrará os requests enviados ao OpenRouter, qual modelo foi usado, erros de autenticação e tokens consumidos. É muito útil para validar se o modelo correto está sendo acionado.

![Logs do OpenRouter mostrando o uso do modelo KAT Coder Pro](/assets/images/openrouter-logs-test.png)
*Exemplo de logs de uso no OpenRouter.*

---

## 11. Extensões: Usando múltiplos providers

O CCR é poderoso e pode agregar vários provedores simultaneamente. Exemplo de configuração avançada:

```json
"Providers": [
  { "name": "openrouter", ... },
  { "name": "ollama", ... },
  { "name": "azure", ... }
]
```

E no roteamento:

```json
"Router": {
  "default": "openrouter,claude-3.5-sonnet",
  "think": "openrouter,claude-3.7-sonnet:thinking",
  "cheap": "ollama,qwen2.5:latest"
}
```

Isso transforma o Claude Code em uma ferramenta multi-LLM unificada.

---

## 12. Conclusão

O **Claude Code** é excepcional como ambiente de desenvolvimento assistido. Ao conectar o **OpenRouter** via **CCR**, ele se torna agnóstico, permitindo o uso de dezenas de modelos diferentes, mantendo seu histórico e fluxo de trabalho.

**Convite:** Teste modelos alternativos via OpenRouter e compartilhe seus resultados comparando Sonnet, Kat, Gemini e DeepSeek dentro da mesma sessão de coding!

### Dica de Ouro: Modelos Gratuitos

Você não precisa gastar nada para começar! O OpenRouter possui uma excelente seleção de modelos gratuitos (Free Tier) que funcionam perfeitamente com o CCR.

![Lista de modelos gratuitos no OpenRouter](/assets/images/openrouter-free-models.png)
*Exemplo de modelos gratuitos disponíveis (como DeepSeek R1 e KAT Coder).*

**[Clique aqui para ver a lista de modelos gratuitos filtrada](https://openrouter.ai/models/?q=free&order=top-weekly){:target="_blank"}**

_O modelo utilizado nos testes deste tutorial foi o KAT Coder PRO V1._
