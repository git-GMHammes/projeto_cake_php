# 🎂 Projeto CakePHP com Docker e NGINX - Windows 11

> **📌 Guia otimizado para Windows 11 usando PowerShell**

## 📋 Estrutura Final do Projeto

```
C:\laragon\www\projeto_cake_php\
│
├── projeto_cake_php\                    # Pasta raiz (já existente)
│   └── cake_php\                        # Aplicação CakePHP
│       ├── bin\
│       ├── config\
│       │   └── app_local.php           # Configurações DB
│       ├── src\
│       ├── templates\
│       ├── webroot\                     # Document root NGINX
│       ├── vendor\
│       ├── composer.json
│       └── ...
│
├── nginx\                               # Configurações NGINX
│   └── default.conf                     # Virtual host
│
└── docker-compose.yml                   # Orquestração Docker
```

---

## 🗺️ ROADMAP DE IMPLEMENTAÇÃO

### **FASE 1: Análise e Backup** 🔍

#### 1.1 - Verificar estrutura atual

**PowerShell:**
```powershell
# Abrir PowerShell como Administrador
C:\laragon\www\projeto_cake_php
Get-ChildItem -Recurse
```

**CMD (alternativa):**
```cmd
C:\laragon\www\projeto_cake_php
dir /s
```

#### 1.2 - Fazer backup completo

**PowerShell:**
```powershell
# Backup com PowerShell (recomendado)
Copy-Item -Path "C:\laragon\www\projeto_cake_php" -Destination "C:\laragon\www\projeto_cake_php_backup" -Recurse -Force
```

**CMD (alternativa):**
```cmd
xcopy "C:\laragon\www\projeto_cake_php" "C:\laragon\www\projeto_cake_php_backup" /E /H /I /Y
```

> **⚠️ IMPORTANTE:** Sempre faça backup antes de reorganizar a estrutura!

#### 1.3 - Verificar Docker Desktop

```powershell
# Verificar se Docker está instalado e rodando
docker --version
docker-compose --version

# Verificar containers em execução
docker ps
```

Se não tiver Docker instalado, baixe em: https://www.docker.com/products/docker-desktop/

---

### **FASE 2: Reorganizar Estrutura CakePHP** 📁

#### 2.1 - Criar pasta interna para o framework

**PowerShell:**
```powershell
C:\laragon\www\projeto_cake_php
New-Item -ItemType Directory -Name "cake_php" -Force
```

**CMD (alternativa):**
```cmd
C:\laragon\www\projeto_cake_php
mkdir cake_php
```

#### 2.2 - Opção A: Mover arquivos existentes

Se já possui arquivos CakePHP na raiz, mova tudo para `cake_php\`:

**PowerShell:**
```powershell
# Navegar até a pasta do projeto
C:\laragon\www\projeto_cake_php

# Mover pastas
Move-Item -Path "src" -Destination "cake_php\" -Force
Move-Item -Path "config" -Destination "cake_php\" -Force
Move-Item -Path "webroot" -Destination "cake_php\" -Force
Move-Item -Path "templates" -Destination "cake_php\" -Force
Move-Item -Path "vendor" -Destination "cake_php\" -Force
Move-Item -Path "bin" -Destination "cake_php\" -Force
Move-Item -Path "plugins" -Destination "cake_php\" -Force
Move-Item -Path "tests" -Destination "cake_php\" -Force
Move-Item -Path "tmp" -Destination "cake_php\" -Force
Move-Item -Path "logs" -Destination "cake_php\" -Force

# Mover arquivos
Move-Item -Path "composer.json" -Destination "cake_php\" -Force
Move-Item -Path "composer.lock" -Destination "cake_php\" -Force
```

**CMD (alternativa):**
```cmd
C:\laragon\www\projeto_cake_php
move src cake_php\
move config cake_php\
move webroot cake_php\
move templates cake_php\
move vendor cake_php\
move composer.json cake_php\
move composer.lock cake_php\
move bin cake_php\
move plugins cake_php\
move tests cake_php\
```

#### 2.3 - Opção B: Instalar CakePHP do zero

**PowerShell/CMD:**
```powershell
C:\laragon\www\projeto_cake_php
composer create-project --prefer-dist cakephp/app:~5.0 cake_php
```

---

### **FASE 3: Configurar Docker** 🐳

#### 3.1 - Criar docker-compose.yml

**Caminho:** `C:\laragon\www\projeto_cake_php\docker-compose.yml`

**PowerShell (criar arquivo):**
```powershell
# Criar arquivo docker-compose.yml
C:\laragon\www\projeto_cake_php
New-Item -ItemType File -Name "docker-compose.yml" -Force
notepad docker-compose.yml
```

**Conteúdo do arquivo:**

```yaml
version: '3.8'

services:
  # ===================================
  # NGINX - Servidor Web
  # ===================================
  nginx:
    image: nginx:alpine
    container_name: cakephp_nginx
    ports:
      - "8080:80"
    volumes:
      - ./projeto_cake_php:/var/www/html
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - php
    networks:
      - cakephp_network
    restart: unless-stopped

  # ===================================
  # PHP-FPM - Processador PHP
  # ===================================
  php:
    image: php:8.2-fpm
    container_name: cakephp_php
    working_dir: /var/www/html
    volumes:
      - ./projeto_cake_php:/var/www/html
    environment:
      - DB_HOST=mysql
      - DB_DATABASE=cakephp_db
      - DB_USERNAME=root
      - DB_PASSWORD=root
    depends_on:
      - mysql
    networks:
      - cakephp_network
    restart: unless-stopped
    command: >
      bash -c "
      docker-php-ext-install pdo pdo_mysql intl mbstring &&
      php-fpm
      "

  # ===================================
  # MySQL - Banco de Dados
  # ===================================
  mysql:
    image: mysql:8.0
    container_name: cakephp_mysql
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: cakephp_db
      MYSQL_USER: cakephp
      MYSQL_PASSWORD: cakephp
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - cakephp_network
    restart: unless-stopped
    command: --default-authentication-plugin=mysql_native_password

  # ===================================
  # PhpMyAdmin - Gerenciador de Banco
  # ===================================
  phpmyadmin:
    image: phpmyadmin/phpmyadmin
    container_name: cakephp_phpmyadmin
    ports:
      - "8081:80"
    environment:
      PMA_HOST: mysql
      PMA_PORT: 3306
      PMA_USER: root
      PMA_PASSWORD: root
    depends_on:
      - mysql
    networks:
      - cakephp_network
    restart: unless-stopped

# ===================================
# Redes
# ===================================
networks:
  cakephp_network:
    driver: bridge

# ===================================
# Volumes Persistentes
# ===================================
volumes:
  mysql_data:
    driver: local
```

#### 3.2 - Entendendo os serviços

| Serviço | Porta | Descrição |
|---------|-------|-----------|
| **nginx** | 8080 | Servidor web que serve a aplicação |
| **php** | 9000 | Processador PHP-FPM |
| **mysql** | 3306 | Banco de dados MySQL |
| **phpmyadmin** | 8081 | Interface web para gerenciar o banco |

---

### **FASE 4: Configurar NGINX** ⚙️

#### 4.1 - Criar pasta nginx

**PowerShell:**
```powershell
C:\laragon\www\projeto_cake_php
New-Item -ItemType Directory -Name "nginx" -Force
```

**CMD (alternativa):**
```cmd
C:\laragon\www\projeto_cake_php
mkdir nginx
```

#### 4.2 - Criar arquivo de configuração

**Caminho:** `C:\laragon\www\nginx\default.conf`

**PowerShell (criar arquivo):**
```powershell
C:\laragon\www\projeto_cake_php\nginx
New-Item -ItemType File -Name "default.conf" -Force
notepad default.conf
```

**Conteúdo do arquivo:**

```nginx
server {
    listen 80;
    server_name localhost;
    
    # Document Root - aponta para webroot do CakePHP
    root /var/www/html/projeto_cake_php/cake_php/webroot;
    index index.php index.html;

    # Logs de acesso e erro
    access_log /var/log/nginx/cakephp_access.log;
    error_log /var/log/nginx/cakephp_error.log;

    # Charset UTF-8
    charset utf-8;

    # Desabilitar logs desnecessários
    location = /favicon.ico { 
        access_log off; 
        log_not_found off; 
    }
    
    location = /robots.txt { 
        access_log off; 
        log_not_found off; 
    }

    # ===================================
    # CakePHP Rewrite Rules
    # ===================================
    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    # ===================================
    # Processar arquivos PHP via PHP-FPM
    # ===================================
    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass php:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
        fastcgi_param PHP_VALUE "upload_max_filesize=100M \n post_max_size=100M";
        fastcgi_buffer_size 128k;
        fastcgi_buffers 256 16k;
        fastcgi_busy_buffers_size 256k;
        fastcgi_temp_file_write_size 256k;
    }

    # ===================================
    # Segurança - Negar acesso a arquivos ocultos
    # ===================================
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }

    # ===================================
    # Cache de arquivos estáticos
    # ===================================
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2|ttf|eot|webp)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    # ===================================
    # Proteger arquivos sensíveis
    # ===================================
    location ~ /(config|tmp|logs|vendor) {
        deny all;
    }
}
```

#### 4.3 - Entendendo a configuração NGINX

- **root**: Aponta para o diretório `webroot` do CakePHP
- **try_files**: Implementa URL amigáveis do CakePHP
- **fastcgi_pass**: Conecta NGINX ao PHP-FPM na porta 9000
- **Cache estático**: Melhora performance com cache de 1 ano para assets
- **Segurança**: Bloqueia acesso a pastas sensíveis

---

### **FASE 5: Configurar CakePHP** 🍰

#### 5.1 - Configurar conexão com banco de dados

**Caminho:** `C:\laragon\www\projeto_cake_php\cake_php\config\app_local.php`

**PowerShell (editar arquivo):**
```powershell
notepad C:\laragon\www\projeto_cake_php\cake_php\config\app_local.php
```

**Conteúdo do arquivo:**

```php
<?php
return [
    /*
     * Debug Level:
     * true: mostra erros detalhados
     * false: modo produção
     */
    'debug' => filter_var(env('DEBUG', true), FILTER_VALIDATE_BOOLEAN),

    /*
     * Security salt usado para hash
     */
    'Security' => [
        'salt' => env('SECURITY_SALT', '__SALT__'),
    ],

    /*
     * Configuração do banco de dados
     */
    'Datasources' => [
        'default' => [
            'className' => 'Cake\Database\Connection',
            'driver' => 'Cake\Database\Driver\Mysql',
            'persistent' => false,
            'host' => env('DB_HOST', 'mysql'),
            'port' => env('DB_PORT', '3306'),
            'username' => env('DB_USERNAME', 'root'),
            'password' => env('DB_PASSWORD', 'root'),
            'database' => env('DB_DATABASE', 'cakephp_db'),
            'encoding' => 'utf8mb4',
            'timezone' => 'UTC',
            'flags' => [],
            'cacheMetadata' => true,
            'log' => false,
            
            /*
             * Para MySQL 5.6+ use o driver nativo
             */
            'quoteIdentifiers' => false,
        ],

        /*
         * Conexão de teste (opcional)
         */
        'test' => [
            'className' => 'Cake\Database\Connection',
            'driver' => 'Cake\Database\Driver\Mysql',
            'persistent' => false,
            'host' => 'mysql',
            'username' => 'root',
            'password' => 'root',
            'database' => 'test_cakephp_db',
            'encoding' => 'utf8mb4',
            'timezone' => 'UTC',
            'cacheMetadata' => true,
            'quoteIdentifiers' => false,
            'log' => false,
        ],
    ],

    /*
     * Configuração de Email (opcional)
     */
    'EmailTransport' => [
        'default' => [
            'className' => 'Smtp',
            'host' => 'localhost',
            'port' => 25,
            'timeout' => 30,
        ],
    ],

    /*
     * Perfis de Email (opcional)
     */
    'Email' => [
        'default' => [
            'transport' => 'default',
            'from' => 'you@localhost',
        ],
    ],
];
```

#### 5.2 - Criar arquivo .env (Opcional)

**Caminho:** `C:\laragon\www\projeto_cake_php\cake_php\.env`

```env
# Configurações de Ambiente
export APP_NAME="CakePHP"
export DEBUG="true"
export APP_ENCODING="UTF-8"
export APP_DEFAULT_LOCALE="pt_BR"
export APP_DEFAULT_TIMEZONE="America/Sao_Paulo"
export SECURITY_SALT="__SALT__"

# Banco de Dados
export DB_HOST="mysql"
export DB_PORT="3306"
export DB_USERNAME="root"
export DB_PASSWORD="root"
export DB_DATABASE="cakephp_db"

# URLs
export APP_FULL_BASE_URL="http://localhost:8080"
```

#### 5.3 - Ajustar permissões (Windows)

**PowerShell (executar como Administrador):**
```powershell
# Navegar até a pasta do CakePHP
cd C:\laragon\www\projeto_cake_php\cake_php
ls

# Dar permissões de escrita para tmp, logs e webroot
icacls "tmp" /grant Everyone:F /T
icacls "logs" /grant Everyone:F /T
icacls "webroot" /grant Everyone:F /T

# Verificar permissões
icacls "tmp"
icacls "logs"
icacls "webroot"
```

> **📝 Nota:** O comando `icacls` é o equivalente Windows ao `chmod` do Linux.

---

### **FASE 6: Iniciar o Ambiente** 🚀

#### 6.1 - Subir os containers Docker

**PowerShell:**
```powershell
# Navegar até a pasta com docker-compose.yml
C:\laragon\www\projeto_cake_php

# Iniciar containers em background
docker-compose up -d

# Ou ver logs durante inicialização
docker-compose up
```

**Flags:**
- `-d`: Executa em background (detached mode)

#### 6.2 - Verificar status dos containers

**PowerShell:**
```powershell
# Ver status de todos os containers
docker-compose ps

# Ver containers em execução
docker ps

# Ver todos os containers (incluindo parados)
docker ps -a
```

**Saída esperada:**
```
NAME                 IMAGE                   STATUS
cakephp_nginx        nginx:alpine            Up
cakephp_php          php:8.2-fpm            Up
cakephp_mysql        mysql:8.0              Up
cakephp_phpmyadmin   phpmyadmin/phpmyadmin  Up
```

#### 6.3 - Verificar logs

**PowerShell:**
```powershell
# Todos os containers
docker-compose logs

# Container específico
docker-compose logs nginx
docker-compose logs php
docker-compose logs mysql

# Seguir logs em tempo real
docker-compose logs -f

# Últimas 50 linhas
docker-compose logs --tail=50
```

#### 6.4 - Acessar a aplicação

Abra seu navegador e acesse:

| URL | Descrição |
|-----|-----------|
| http://localhost:8080 | **Aplicação CakePHP** |
| http://localhost:8081 | **PhpMyAdmin** (gerenciar banco) |

---

### **FASE 7: Instalar Dependências PHP** 📦

#### 7.1 - Entrar no container PHP

**PowerShell:**
```powershell
# Entrar no container PHP via bash
docker exec -it cakephp_php bash

# Ou via sh (Alpine Linux)
docker exec -it cakephp_php sh
```

#### 7.2 - Instalar extensões PHP necessárias

**Dentro do container:**
```bash
# Instalar extensões
docker-php-ext-install pdo pdo_mysql intl mbstring

# Reiniciar PHP-FPM
kill -USR2 1
```

**Ou fazer tudo de uma vez via PowerShell (sem entrar no container):**
```powershell
docker exec cakephp_php docker-php-ext-install pdo pdo_mysql intl mbstring
docker-compose restart php
```

#### 7.3 - Instalar dependências do Composer

**PowerShell (executar de fora do container):**
```powershell
docker exec -it cakephp_php composer install -d /var/www/html/projeto_cake_php/cake_php
```

**Ou dentro do container:**
```bash
docker exec -it cakephp_php bash
cd /var/www/html/projeto_cake_php/cake_php
composer install
```

#### 7.4 - Verificar instalação do CakePHP

**PowerShell:**
```powershell
docker exec cakephp_php php /var/www/html/projeto_cake_php/cake_php/bin/cake.php version
```

**Saída esperada:**
```
CakePHP 5.x.x
```

#### 7.5 - Sair do container

```bash
exit
```

---

### **FASE 8: Testes e Validação** ✅

#### 8.1 - Testar conexão com o banco

**PowerShell:**
```powershell
# Testar se CakePHP está funcionando
Start-Process "http://localhost:8080"
```

Você deve ver a página inicial do CakePHP com:
- ✅ **Environment**: OK
- ✅ **Filesystem**: OK
- ✅ **Database**: OK (se configurado corretamente)

#### 8.2 - Verificar logs do NGINX

**PowerShell:**
```powershell
docker logs cakephp_nginx

# Ou seguir logs em tempo real
docker logs -f cakephp_nginx
```

#### 8.3 - Verificar logs do PHP

**PowerShell:**
```powershell
docker logs cakephp_php

# Ou seguir logs em tempo real
docker logs -f cakephp_php
```

#### 8.4 - Testar PhpMyAdmin

**PowerShell:**
```powershell
Start-Process "http://localhost:8081"
```

1. **Usuário:** `root`
2. **Senha:** `root`
3. Verifique se o banco `cakephp_db` existe

#### 8.5 - Executar comandos CakePHP

**PowerShell:**
```powershell
# Listar rotas
docker exec cakephp_php php /var/www/html/projeto_cake_php/cake_php/bin/cake.php routes

# Criar migration
docker exec cakephp_php php /var/www/html/projeto_cake_php/cake_php/bin/cake.php bake migration CreateUsers

# Gerar model/controller/views
docker exec cakephp_php php /var/www/html/projeto_cake_php/cake_php/bin/cake.php bake all users
```

---

## 🔧 Comandos Úteis do Docker (PowerShell)

### Gerenciamento básico

```powershell
# Iniciar containers
docker-compose up -d

# Parar containers
docker-compose stop

# Parar e remover containers
docker-compose down

# Reiniciar containers
docker-compose restart

# Reiniciar container específico
docker-compose restart php
docker-compose restart nginx

# Rebuild containers (após mudanças no Dockerfile)
docker-compose up -d --build

# Ver logs em tempo real
docker-compose logs -f

# Ver logs de um serviço específico
docker-compose logs -f nginx
docker-compose logs -f php
docker-compose logs -f mysql

# Ver últimas 100 linhas de log
docker-compose logs --tail=100
```

### Gerenciamento de volumes

```powershell
# Listar volumes
docker volume ls

# Inspecionar volume específico
docker volume inspect projeto_cake_php_mysql_data

# Remover volumes não utilizados
docker volume prune

# Remover tudo (containers, volumes, networks)
docker-compose down -v
```

### Executar comandos nos containers

```powershell
# Entrar no container PHP
docker exec -it cakephp_php bash

# Entrar no container MySQL
docker exec -it cakephp_mysql mysql -u root -proot

# Executar comando sem entrar no container
docker exec cakephp_php php /var/www/html/projeto_cake_php/cake_php/bin/cake.php version

# Copiar arquivo do Windows para container
docker cp C:\meu_arquivo.txt cakephp_php:/var/www/html/

# Copiar arquivo do container para Windows
docker cp cakephp_php:/var/www/html/arquivo.txt C:\destino\
```

---

## 🎯 Comandos Úteis do CakePHP (PowerShell)

### Bake (Gerador de código)

```powershell
# Gerar tudo (model, controller, views)
docker exec cakephp_php php /var/www/html/projeto_cake_php/cake_php/bin/cake.php bake all users

# Gerar apenas model
docker exec cakephp_php php /var/www/html/projeto_cake_php/cake_php/bin/cake.php bake model Users

# Gerar apenas controller
docker exec cakephp_php php /var/www/html/projeto_cake_php/cake_php/bin/cake.php bake controller Users

# Gerar apenas views (templates)
docker exec cakephp_php php /var/www/html/projeto_cake_php/cake_php/bin/cake.php bake template Users
```

### Migrations

```powershell
# Criar nova migration
docker exec cakephp_php php /var/www/html/projeto_cake_php/cake_php/bin/cake.php bake migration CreateUsers

# Executar migrations
docker exec cakephp_php php /var/www/html/projeto_cake_php/cake_php/bin/cake.php migrations migrate

# Rollback da última migration
docker exec cakephp_php php /var/www/html/projeto_cake_php/cake_php/bin/cake.php migrations rollback

# Ver status das migrations
docker exec cakephp_php php /var/www/html/projeto_cake_php/cake_php/bin/cake.php migrations status
```

### Cache

```powershell
# Limpar todo o cache
docker exec cakephp_php php /var/www/html/projeto_cake_php/cake_php/bin/cake.php cache clear_all

# Limpar cache específico
docker exec cakephp_php php /var/www/html/projeto_cake_php/cake_php/bin/cake.php cache clear _cake_model_
docker exec cakephp_php php /var/www/html/projeto_cake_php/cake_php/bin/cake.php cache clear _cake_core_
```

### Debug e Rotas

```powershell
# Listar todas as rotas
docker exec cakephp_php php /var/www/html/projeto_cake_php/cake_php/bin/cake.php routes

# Verificar versão
docker exec cakephp_php php /var/www/html/projeto_cake_php/cake_php/bin/cake.php version
```

---

## 🐛 Solução de Problemas Comuns (Windows)

### Problema 1: Erro 502 Bad Gateway

**Causa:** PHP-FPM não está respondendo

**Solução (PowerShell):**
```powershell
# Verificar logs do PHP
docker logs cakephp_php

# Reiniciar container PHP
docker-compose restart php

# Verificar se o PHP está rodando
docker exec cakephp_php ps aux
```

### Problema 2: Erro de permissão em tmp/ ou logs/

**Causa:** Pasta sem permissão de escrita

**Solução (PowerShell como Administrador):**
```powershell
# Ajustar permissões no Windows
C:\laragon\www\projeto_cake_php\cake_php
icacls "tmp" /grant Everyone:F /T
icacls "logs" /grant Everyone:F /T

# Dentro do container
docker exec -it cakephp_php bash
cd /var/www/html/projeto_cake_php/cake_php
chmod -R 777 tmp/
chmod -R 777 logs/
chown -R www-data:www-data tmp/
chown -R www-data:www-data logs/
```

### Problema 3: Banco de dados não conecta

**Causa:** Configurações incorretas em app_local.php

**Solução (PowerShell):**
```powershell
# Verificar se o MySQL está rodando
docker exec cakephp_mysql mysql -u root -proot -e "SHOW DATABASES;"

# Testar conexão
docker exec cakephp_php php /var/www/html/projeto_cake_php/cake_php/bin/cake.php server check
```

### Problema 4: CSS/JS não carrega

**Causa:** Caminho incorreto ou permissões

**Solução (PowerShell):**
```powershell
# Verificar se webroot está acessível
docker exec cakephp_php ls -la /var/www/html/projeto_cake_php/cake_php/webroot/

# Ajustar permissões no Windows
C:\laragon\www\projeto_cake_php\cake_php
icacls "webroot" /grant Everyone:R /T
```

### Problema 5: Container não inicia

**Causa:** Porta já em uso

**Solução (PowerShell como Administrador):**
```powershell
# Verificar portas em uso
netstat -ano | findstr :8080
netstat -ano | findstr :3306

# Identificar processo (PID na última coluna)
Get-Process -Id <PID>

# Matar processo
Stop-Process -Id <PID> -Force

# Ou usar taskkill
taskkill /PID <PID> /F

# Alternativamente, mudar porta no docker-compose.yml
# ports:
#   - "8090:80"  # Mudar de 8080 para 8090
```

### Problema 6: Docker Desktop não inicia

**Solução (PowerShell como Administrador):**
```powershell
# Reiniciar serviço do Docker
Restart-Service docker

# Verificar status
Get-Service docker

# Verificar WSL 2 (necessário para Docker Desktop)
wsl --list --verbose
wsl --update

# Verificar Hyper-V (necessário para Docker Desktop)
Get-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V
```

### Problema 7: Erro de permissão ao criar arquivos

**Causa:** Docker Desktop precisa de acesso às pastas

**Solução:**
1. Abra **Docker Desktop**
2. Vá em **Settings** > **Resources** > **File Sharing**
3. Adicione `C:\laragon\www` às pastas compartilhadas
4. Clique em **Apply & Restart**

---

## 📂 Estrutura de Pastas do CakePHP

```
cake_php\
├── bin\                    # Executáveis (cake, cake.php)
├── config\                 # Arquivos de configuração
│   ├── app.php            # Configurações principais
│   ├── app_local.php      # Configurações locais (DB)
│   ├── bootstrap.php      # Bootstrap da aplicação
│   └── routes.php         # Definição de rotas
├── logs\                   # Logs da aplicação
├── plugins\                # Plugins instalados
├── resources\              # Assets não compilados
├── src\                    # Código fonte da aplicação
│   ├── Controller\        # Controllers
│   ├── Model\             # Models (Table e Entity)
│   ├── View\              # Views e Helpers
│   └── Application.php    # Classe principal
├── templates\              # Templates (views)
│   ├── layout\            # Layouts
│   ├── element\           # Elementos reutilizáveis
│   └── Pages\             # Páginas estáticas
├── tests\                  # Testes automatizados
├── tmp\                    # Arquivos temporários e cache
│   ├── cache\             # Cache da aplicação
│   └── sessions\          # Sessões
├── vendor\                 # Dependências do Composer
├── webroot\                # Document root público
│   ├── css\               # Arquivos CSS
│   ├── js\                # Arquivos JavaScript
│   ├── img\               # Imagens
│   └── index.php          # Front controller
├── .gitignore
├── composer.json           # Dependências PHP
└── composer.lock
```

---

## 🪟 Comandos PowerShell Avançados

### Scripts úteis

```powershell
# Script para limpar Docker (limpa_docker.ps1)
function Limpar-Docker {
    Write-Host "Parando containers..."
    docker-compose down
    
    Write-Host "Removendo containers órfãos..."
    docker container prune -f
    
    Write-Host "Removendo imagens não utilizadas..."
    docker image prune -a -f
    
    Write-Host "Removendo volumes não utilizados..."
    docker volume prune -f
    
    Write-Host "Removendo networks não utilizadas..."
    docker network prune -f
    
    Write-Host "Limpeza concluída!"
}

# Executar: Limpar-Docker
```

```powershell
# Script para resetar ambiente (reset_ambiente.ps1)
function Reset-Ambiente {
    Write-Host "Parando containers..."
    docker-compose down -v
    
    Write-Host "Removendo pasta tmp e logs..."
    Remove-Item -Path "C:\laragon\www\projeto_cake_php\cake_php\tmp\*" -Recurse -Force
    Remove-Item -Path "C:\laragon\www\projeto_cake_php\cake_php\logs\*" -Recurse -Force
    
    Write-Host "Recriando pastas..."
    New-Item -ItemType Directory -Path "C:\laragon\www\projeto_cake_php\cake_php\tmp\cache" -Force
    New-Item -ItemType Directory -Path "C:\laragon\www\projeto_cake_php\cake_php\tmp\sessions" -Force
    
    Write-Host "Ajustando permissões..."
    icacls "C:\laragon\www\projeto_cake_php\cake_php\tmp" /grant Everyone:F /T
    icacls "C:\laragon\www\projeto_cake_php\cake_php\logs" /grant Everyone:F /T
    
    Write-Host "Iniciando containers..."
    docker-compose up -d
    
    Write-Host "Ambiente resetado!"
}

# Executar: Reset-Ambiente
```

---

## 📚 Recursos Adicionais

### Documentação Oficial

- **CakePHP:** https://book.cakephp.org/5/pt/
- **Docker Desktop (Windows):** https://docs.docker.com/desktop/windows/
- **Docker Compose:** https://docs.docker.com/compose/
- **NGINX:** https://nginx.org/en/docs/
- **PHP-FPM:** https://www.php.net/manual/pt_BR/install.fpm.php
- **PowerShell:** https://docs.microsoft.com/pt-br/powershell/

### Ferramentas Recomendadas para Windows

- **Windows Terminal:** https://aka.ms/terminal
- **Visual Studio Code:** https://code.visualstudio.com/
- **Docker Desktop:** https://www.docker.com/products/docker-desktop/
- **Git for Windows:** https://gitforwindows.org/
- **Composer:** https://getcomposer.org/download/

### Extensões VSCode Úteis

- Docker
- PHP Intelephense
- CakePHP Snippets
- GitLens
- PowerShell

---

## ✅ Checklist de Implementação

Use esta lista para acompanhar o progresso:

- [ ] **Pré-requisitos:**
  - [ ] Docker Desktop instalado
  - [ ] WSL 2 configurado (se necessário)
  - [ ] Composer instalado
  - [ ] PowerShell 5.1+ ou Windows Terminal
- [ ] **Fase 1:** Backup realizado
- [ ] **Fase 2:** Estrutura CakePHP reorganizada
- [ ] **Fase 3:** `docker-compose.yml` criado
- [ ] **Fase 4:** `nginx\default.conf` configurado
- [ ] **Fase 5:** `app_local.php` atualizado
- [ ] **Fase 6:** Containers Docker iniciados
- [ ] **Fase 7:** Dependências PHP instaladas
- [ ] **Fase 8:** Testes validados
- [ ] ✅ Aplicação acessível em http://localhost:8080
- [ ] ✅ PhpMyAdmin acessível em http://localhost:8081
- [ ] ✅ Banco de dados conectado
- [ ] ✅ Permissões ajustadas (tmp, logs, webroot)

---

## 💡 Dicas Importantes para Windows

### Docker Desktop

1. **Certifique-se que Docker Desktop está rodando** antes de executar comandos
2. **Compartilhe drives** nas configurações do Docker Desktop
3. **WSL 2** é recomendado para melhor performance
4. **Hyper-V** deve estar habilitado (Windows 10/11 Pro)

### PowerShell

1. **Execute como Administrador** para comandos de permissão
2. **Windows Terminal** oferece melhor experiência que CMD
3. Use **Tab** para autocompletar comandos
4. `Ctrl+C` para cancelar comandos em execução

### Portas

Se as portas padrão estiverem em uso:
- **8080** → pode mudar para 8090, 8000, 3000
- **3306** → pode mudar para 3307, 3308
- **8081** → pode mudar para 8082, 9000

### Performance

Para melhor performance no Windows:
1. Use **WSL 2** ao invés de Hyper-V
2. Coloque projetos dentro do **filesystem WSL** (mais rápido)
3. **Desabilite antivírus** na pasta do projeto (pode deixar lento)
4. **Aumente RAM** dedicada ao Docker Desktop (Settings > Resources)

---

## 🤝 Contribuindo

Se encontrar algum problema ou tiver sugestões de melhorias:

1. Documente o problema encontrado
2. Descreva a solução aplicada
3. Atualize este README.md

---

## 📝 Notas Importantes

- **Ambiente de Desenvolvimento:** Esta configuração é otimizada para desenvolvimento, não para produção
- **Senhas Padrão:** Altere as senhas antes de usar em produção
- **Performance:** Para melhor performance em produção, considere usar volumes nomeados ao invés de bind mounts
- **SSL/HTTPS:** Para ambiente de produção, configure certificados SSL
- **Backup:** Sempre faça backup regular do banco de dados
- **Firewall:** Pode ser necessário liberar portas no firewall do Windows
- **Antivírus:** Alguns antivírus podem bloquear Docker, adicione exceção se necessário

---

## 📧 Suporte

Para dúvidas ou problemas:

- Verifique a seção **Solução de Problemas**
- Consulte os logs dos containers (`docker logs`)
- Acesse a documentação oficial do CakePHP
- Verifique logs do Docker Desktop

---

## 🔗 Links Rápidos

- **CakePHP Docs:** https://book.cakephp.org/5/pt/
- **Docker Windows:** https://docs.docker.com/desktop/windows/
- **PowerShell Docs:** https://docs.microsoft.com/powershell/

---

**Desenvolvido com ❤️ usando CakePHP, Docker e NGINX no Windows 11**

**Última atualização:** Janeiro 2026

**Sistema Operacional:** Windows 11  
**Shell:** PowerShell 5.1+