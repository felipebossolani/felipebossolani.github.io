---
layout: post
title: "Como instalar Postgres no Coolify (+PgVector e PostGIS)"
date: 2025-12-08 20:30:00 -0300
categories: [Coolify, Postgres, Tutorial, DevOps]
tags: [database, self-hosting, docker, postgres]
---

**Construindo em público**

Este artigo faz parte da minha jornada documentando a construção de uma infraestrutura de dados moderna e independente. Se você está chegando agora, vale a pena conferir os capítulos anteriores dessa saga:

*   [Minha primeira VPS na Hetzner](/posts/minha-primeira-vps-na-hetzner/) — Onde tudo começou (setup inicial e segurança).
*   [Instalando e primeiros passos com o Coolify](/posts/comecando-com-coolify-na-vps/) — Configurando nossa PaaS open-source.
*   [Refatorando o pipeline da B3](/posts/refatorando-pipeline-b3-cloudflare-r2/) — A arquitetura serverless que vai consumir este banco.

---

Na última semana, focamos na ingestão de dados usando Cloudflare R2. Agora, precisamos de um destino robusto para processar e servir essas informações.

Como optei pelo caminho do *self-hosting* para manter o controle total (e custos baixos), preciso de uma ferramenta que tire a dor de cabeça de gerenciar bancos de dados em produção.

É aqui que o **[Coolify](https://coolify.io){:target="_blank"}** brilha novamente. Ele simplifica drasticamente a criação de serviços "Cloud Native". Neste tutorial, vamos subir uma instância do PostgreSQL (com suporte a IA/Vetores) em poucos cliques.

## 1. Acesse seu Projeto

No dashboard principal do Coolify, navegue até a aba **Projects**. Selecione o projeto onde deseja implantar seu banco de dados.

![Lista de Projetos no Coolify](/assets/files/2025-12-08-postgres-coolify/01-projects-list.png)
*Painel de projetos do Coolify.*

Selecione o ambiente desejado (geralmente "Production" ou "Development").

![Seleção do Projeto](/assets/files/2025-12-08-postgres-coolify/02-select-project.png)
*Dashboard do projeto selecionado.*

---

## 2. Adicione um Novo Recurso

Dentro do ambiente do seu projeto, clique no botão **+ Add Resource**.

Você verá uma tela com opções para diferentes tipos de recursos (Applications, Databases, Docker, Services).

![Menu "New Resource"](/assets/files/2025-12-08-postgres-coolify/03-new-resource-menu.png)
*Menu de seleção de novo recurso.*

Para instalar o Postgres, você pode usar a barra de busca "Type / to search..." e digitar "Postgres", ou navegar até a seção de **Databases**.

---

## 3. Escolha o "Sabor" do Postgres

O Coolify oferece versões pré-configuradas do PostgreSQL para diferentes necessidades. Ao selecionar Postgres, você verá as seguintes opções:

![Seleção de Tipo de Postgres](/assets/files/2025-12-08-postgres-coolify/04-postgres-flavors.png)
*Opções de instalação do PostgreSQL.*

*   **PostgreSQL 17 (default):** A versão padrão e robusta do banco relacional, sem extensões extras pré-instaladas. **Esta é a opção recomendada** para a maioria dos casos de uso web tradicionais.
*   **Supabase PostgreSQL (with extensions):** Uma versão "turbinada" que imita o backend do Supabase, já incluindo diversas extensões úteis.
*   **PostGIS (AMD only):** Essencial se você trabalha com dados geoespaciais (mapas, coordenadas). *Nota: Verifique a compatibilidade com a arquitetura do seu servidor (AMD64 vs ARM64).*
*   **PGVector (17):** A escolha perfeita para aplicações de IA e RAG (Retrieval-Augmented Generation), pois adiciona suporte nativo a vetores embeddings.

Clique na opção que melhor se adapta ao seu projeto. Para este tutorial, seguiremos com a versão padrão ou PGVector.

---

## 4. Defina o Destino (Servidor)

Após selecionar o banco de dados, o Coolify perguntará onde você deseja fazer o deploy.

![Seleção de Servidor/Network](/assets/files/2025-12-08-postgres-coolify/05-destination-selection.png)
*Escolha do servidor de destino e rede Docker.*

Selecione o servidor (ex: `localhost` ou um VPS remoto conectado) e a respectiva rede Docker (`coolify` é o padrão) onde o banco deve rodar.

---

---

## 5. Configuração Geral

Agora você está na tela principal de configuração do seu novo banco de dados. Aqui você pode definir os detalhes cruciais:

*   **Name:** Dê um nome amigável para identificar seu serviço (ex: `postgres-financial-market`).
*   **Description:** Uma breve descrição do propósito do banco.
*   **Image:** A imagem Docker que será usada. O padrão geralmente é `postgres:17-alpine` (versão leve).
*   **Credentials:** O usuário padrão (`postgres`) e a senha gerada automaticamente (que você pode visualizar clicando no ícone de olho). **Guarde esta senha!**

![Tela de Configuração do Postgres](/assets/files/2025-12-08-postgres-coolify/06-configuration-overview.png)
*Painel de configuração geral. Note o botão "Start" no canto superior direito.*

Se precisar ajustar variáveis de ambiente ou portas, explore as abas abaixo ("Environment Variables", "Network").

---

## 6. Armazenamento Persistente (Storage)

É fundamental garantir que seus dados não sejam perdidos caso o container reinicie. Vá até a aba **Persistent Storage**.

![Configuração de Volumes](/assets/files/2025-12-08-postgres-coolify/07-persistent-storage.png)
*Verificação dos volumes persistentes.*

O Coolify cria automaticamente um volume (ex: `postgres-financial-market-volume`) mapeado para o diretório de dados do Postgres (`/var/lib/postgresql/data`). Normalmente você não precisa alterar isso, mas é bom conferir se está lá.

---

## 7. Iniciar o Serviço

Tudo pronto? Volte para a aba **Configuration** (ou clique no botão no topo) e pressione **Start**.

O Coolify iniciará o processo de deploy:
1.  Baixar a imagem Docker (Pull).
2.  Criar os volumes.
3.  Iniciar o container.

Você pode acompanhar tudo em tempo real clicando em **Logs**.

![Logs de Deploy](/assets/files/2025-12-08-postgres-coolify/08-deployment-logs.png)
*Logs mostrando o sucesso do deploy ("Database started").*

Assim que ver a mensagem `Database started`, seu Postgres está no ar e pronto para receber conexões!

---

## Conclusão

Em poucos minutos, você subiu uma instância robusta de PostgreSQL. Agora você pode conectar suas aplicações usando as credenciais geradas ou configurar backups automáticos na aba **Backups**.

