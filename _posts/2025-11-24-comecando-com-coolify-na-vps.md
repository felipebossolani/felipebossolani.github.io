---
layout: post
title: "Instalando e primeiros passos com o Coolify na VPS"
date: 2025-11-24 22:47:00 -0300
categories: [DevOps, Coolify]
tags: [coolify, vps, docker, deploy]
---

Após concluir a configuração de segurança e diagnóstico da VPS (**[Fase 1]({% post_url 2025-11-23-minha-primeira-vps-na-hetzner %})**), iniciei a instalação do **[Coolify](https://coolify.io){:target="_blank"}** – minha plataforma de deploy para os side-projects que pretendo desenvolver ao longo de 2026.

O objetivo da Fase 2 é instalar o Coolify com segurança, validar sua execução e deixá-lo pronto para receber os primeiros deploys reais.

---

## 1. Abrindo apenas as portas necessárias

Na Fase 1, o firewall (UFW) estava liberando apenas o SSH.
Antes da instalação, liberei somente o mínimo necessário para o Coolify funcionar:

```bash
sudo ufw allow 22/tcp      # SSH
sudo ufw allow 80/tcp      # HTTP
sudo ufw allow 443/tcp     # HTTPS
sudo ufw allow 8000/tcp    # Painel do Coolify
sudo ufw allow 6001/tcp    # Realtime
sudo ufw allow 6002/tcp    # Realtime
sudo ufw status
```

---

## 2. Instalação oficial do Coolify

Usei o script oficial de instalação, conforme a **[vasta documentação](https://coolify.io/docs/){:target="_blank"}**:

```bash
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | sudo bash
```

O script instalou Docker e Docker Compose, baixou as imagens necessárias e configurou o Coolify como serviço do sistema.

---

## 3. Backup obrigatório do arquivo .env

Ao final, o instalador exibe um aviso importante:

> It is highly recommended to backup your Environment variables file (/data/coolify/source/.env) to a safe location, outside of this server.

Localizei o arquivo com:

```bash
sudo find /data -name ".env"
```

Depois gerei uma cópia segura e transferi para minha máquina local:

```bash
sudo cp /data/coolify/source/.env /home/me/coolify.env
sudo chown me:me /home/me/coolify.env
scp -i ~/.ssh/id_ed25519 me@IP:/home/me/coolify.env ./coolify-backup.env
```

O conteúdo foi armazenado como Secure Note no Bitwarden.
Este arquivo é essencial para restaurar o Coolify em outro servidor.

---

## 4. Acesso ao painel do Coolify

Com o Coolify rodando, acessei pelo navegador:

`http://IP:8000`

Na primeira vez que se acessa, é obrigatório criar o usuário administrador.
Guardei e-mail e senha também no Bitwarden.

![Coolify Register](/assets/images/coolify-install/coolify-01.png)

---

## 5. Criando o destino de deploy

Após criar o admin, o próximo passo é selecionar o tipo de servidor.

![Coolify Welcome](/assets/images/coolify-install/coolify-02.png)

Como o Coolify foi instalado na própria VPS, escolhi:

**This Machine (Quick Start)**

![Coolify Choose Server](/assets/images/coolify-install/coolify-03.png)

No momento de testar a conexão, apareceu o erro:

```text
sudo: a password is required
sudo: a terminal is required to read the password
```

![Coolify Connection Error](/assets/images/coolify-install/coolify-04.png)

Isso aconteceu porque o Coolify tenta se conectar via SSH e executar comandos com sudo, mas não tem terminal interativo.
A solução foi permitir sudo sem senha para o usuário `me`, com segurança, pois o SSH por senha e login root já estavam bloqueados desde a Fase 1.

```bash
sudo visudo
```

Adicionei no final do arquivo:

```text
me ALL=(ALL) NOPASSWD:ALL
```

Após isso, o Coolify passou a validar a conexão corretamente.

---

## 6. Setup finalizado

Com a conexão validada, o Coolify configurou automaticamente:
*   servidor local (Docker host)
*   primeiro projeto padrão
*   verificação completa do Docker Engine

Tela final:

```text
Setup Complete!
Server: localhost
Project: My first project
Docker Engine: installed and running
```

![Coolify Setup Complete](/assets/images/coolify-install/coolify-05.png)

A Fase 2 se encerra aqui:
Coolify instalado, seguro, conectado e pronto para receber o primeiro deploy real.


