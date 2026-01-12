# 🎂 Projeto CakePHP com Docker e NGINX

## 📋 Estrutura Final do Projeto

```
C:\laragon\www\projeto_cake_php\
│
├── projeto_cake_php/                    # Pasta raiz (já existente)
│   └── cake_php/                        # Aplicação CakePHP
│       ├── bin/
│       ├── config/
│       │   └── app_local.php           # Configurações DB
│       ├── src/
│       ├── templates/
│       ├── webroot/                     # Document root NGINX
│       ├── vendor/
│       ├── composer.json
│       └── ...
│
├── nginx/                               # Configurações NGINX
│   └── default.conf                     # Virtual host
│
└── docker-compose.yml                   # Orquestração Docker
```

---

## 🗺️ ROADMAP DE IMPLEMENTAÇÃO

### **FASE 1: Análise e Backup** 🔍

#### 1.1 - Verificar estrutura atual
```bash
cd C:\laragon\www\projeto_cake_php\projeto_cake_php
dir /s
```

#### 1.2 - Fazer backup completo
```bash
# No PowerShell ou CMD
xcopy "C:\laragon\www\projeto_cake_php\projeto_cake_php" "C:\laragon\www\projeto_cake_php\projeto_cake_php_backup" /E /H /I
```

> **⚠️ IMPORTANTE:** Sempre faça backup antes de reorganizar a estrutura!

---

### **FASE 2: Reorganizar Estrutura CakePHP** 📁

#### 2.1 - Criar pasta interna para o framework
```bash
cd C:\laragon\www\projeto_cake_php\projeto_cake_php
mkdir cake_php
```

#### 2.2 - Opção A: Mover arquivos existentes

Se já possui arquivos CakePHP na raiz, mova tudo para `cake_php/`:

```bash
# Mover todos os arquivos para a pasta cake_php
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

```bash
cd C:\laragon\www\projeto_cake_php\projeto_cake_php
composer create-project --prefer-dist cakephp/app:~5.0 cake_php
```

---

### **FASE 3: Configurar Docker** 🐳

#### 3.1 - Criar docker-compose.yml

**Caminho:** `C:\laragon\www\projeto_cake_php\projeto_cake_php\docker-compose.yml`

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
```bash
cd C:\laragon\www\projeto_cake_php
mkdir nginx
```

#### 4.2 - Criar arquivo de configuração

**Caminho:** `C:\laragon\www\projeto_cake_php\nginx\default.conf`

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

**Caminho:** `C:\laragon\www\projeto_cake_php\projeto_cake_php\cake_php\config\app_local.php`

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

**Caminho:** `C:\laragon\www\projeto_cake_php\projeto_cake_php\cake_php\.env`

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

#### 5.3 - Ajustar permissões (Importante!)

No **Git Bash** ou **WSL**:

```bash
cd /c/laragon/www/projeto_cake_php/cake_php
chmod -R 775 tmp/
chmod -R 775 logs/
chmod -R 775 webroot/
```

No **PowerShell** (Windows):

```powershell
cd C:\laragon\www\projeto_cake_php\projeto_cake_php\cake_php
icacls tmp /grant Everyone:F /T
icacls logs /grant Everyone:F /T
icacls webroot /grant Everyone:F /T
```

---

### **FASE 6: Iniciar o Ambiente** 🚀

#### 6.1 - Subir os containers Docker

```bash
cd C:\laragon\www\projeto_cake_php
docker-compose up -d
```

**Flags:**
- `-d`: Executa em background (detached mode)

#### 6.2 - Verificar status dos containers

```bash
docker-compose ps
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

```bash
# Todos os containers
docker-compose logs

# Container específico
docker-compose logs nginx
docker-compose logs php
docker-compose logs mysql

# Seguir logs em tempo real
docker-compose logs -f
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

```bash
docker exec -it cakephp_php bash
```

#### 7.2 - Instalar extensões PHP necessárias

```bash
# Dentro do container
docker-php-ext-install pdo pdo_mysql intl mbstring

# Reiniciar o PHP-FPM
kill -USR2 1
```

#### 7.3 - Instalar dependências do Composer

```bash
# Dentro do container
cd /var/www/html/projeto_cake_php/cake_php
composer install
```

#### 7.4 - Verificar instalação do CakePHP

```bash
bin/cake version
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

Acesse: http://localhost:8080

Você deve ver a página inicial do CakePHP com:
- ✅ **Environment**: OK
- ✅ **Filesystem**: OK
- ✅ **Database**: OK (se configurado corretamente)

#### 8.2 - Verificar logs do NGINX

```bash
docker logs cakephp_nginx
```

#### 8.3 - Verificar logs do PHP

```bash
docker logs cakephp_php
```

#### 8.4 - Testar PhpMyAdmin

1. Acesse: http://localhost:8081
2. **Usuário:** `root`
3. **Senha:** `root`
4. Verifique se o banco `cakephp_db` existe

#### 8.5 - Executar comandos CakePHP

```bash
# Entrar no container
docker exec -it cakephp_php bash

# Listar rotas
cd /var/www/html/projeto_cake_php/cake_php
bin/cake routes

# Criar migration
bin/cake bake migration CreateUsers

# Criar model/controller/views
bin/cake bake all users
```

---

## 🔧 Comandos Úteis do Docker

### Gerenciamento básico

```bash
# Iniciar containers
docker-compose up -d

# Parar containers
docker-compose stop

# Parar e remover containers
docker-compose down

# Reiniciar containers
docker-compose restart

# Rebuild containers (após mudanças no Dockerfile)
docker-compose up -d --build

# Ver logs em tempo real
docker-compose logs -f

# Ver logs de um serviço específico
docker-compose logs -f nginx
docker-compose logs -f php
docker-compose logs -f mysql
```

### Gerenciamento de volumes

```bash
# Listar volumes
docker volume ls

# Remover volumes não utilizados
docker volume prune

# Remover tudo (containers, volumes, networks)
docker-compose down -v
```

### Executar comandos nos containers

```bash
# Entrar no container PHP
docker exec -it cakephp_php bash

# Entrar no container MySQL
docker exec -it cakephp_mysql mysql -u root -p

# Executar comando sem entrar no container
docker exec cakephp_php php /var/www/html/projeto_cake_php/cake_php/bin/cake.php version
```

---

## 🎯 Comandos Úteis do CakePHP

### Bake (Gerador de código)

```bash
# Entrar no container
docker exec -it cakephp_php bash
cd /var/www/html/projeto_cake_php/cake_php

# Gerar tudo (model, controller, views)
bin/cake bake all users

# Gerar apenas model
bin/cake bake model Users

# Gerar apenas controller
bin/cake bake controller Users

# Gerar apenas views (templates)
bin/cake bake template Users
```

### Migrations

```bash
# Criar nova migration
bin/cake bake migration CreateUsers

# Executar migrations
bin/cake migrations migrate

# Rollback da última migration
bin/cake migrations rollback

# Ver status das migrations
bin/cake migrations status
```

### Cache

```bash
# Limpar cache
bin/cake cache clear_all

# Limpar cache específico
bin/cake cache clear _cake_model_
bin/cake cache clear _cake_core_
```

### Debug e Rotas

```bash
# Listar todas as rotas
bin/cake routes

# Modo debug
bin/cake server --debug
```

---

## 🐛 Solução de Problemas Comuns

### Problema 1: Erro 502 Bad Gateway

**Causa:** PHP-FPM não está respondendo

**Solução:**
```bash
# Verificar logs do PHP
docker logs cakephp_php

# Reiniciar container PHP
docker-compose restart php

# Verificar se o PHP está rodando
docker exec -it cakephp_php ps aux | grep php
```

### Problema 2: Erro de permissão em tmp/ ou logs/

**Causa:** Pasta sem permissão de escrita

**Solução:**
```bash
# Entrar no container
docker exec -it cakephp_php bash

# Ajustar permissões
cd /var/www/html/projeto_cake_php/cake_php
chmod -R 777 tmp/
chmod -R 777 logs/
chown -R www-data:www-data tmp/
chown -R www-data:www-data logs/
```

### Problema 3: Banco de dados não conecta

**Causa:** Configurações incorretas em app_local.php

**Solução:**
```bash
# Verificar se o MySQL está rodando
docker exec -it cakephp_mysql mysql -u root -proot -e "SHOW DATABASES;"

# Testar conexão manualmente
docker exec -it cakephp_php bash
cd /var/www/html/projeto_cake_php/cake_php
bin/cake server check
```

### Problema 4: CSS/JS não carrega

**Causa:** Caminho incorreto ou permissões

**Solução:**
```bash
# Verificar se webroot está acessível
docker exec -it cakephp_php bash
ls -la /var/www/html/projeto_cake_php/cake_php/webroot/

# Ajustar permissões
chmod -R 755 /var/www/html/projeto_cake_php/cake_php/webroot/
```

### Problema 5: Container não inicia

**Causa:** Porta já em uso

**Solução:**
```bash
# Verificar portas em uso (Windows PowerShell)
netstat -ano | findstr :8080
netstat -ano | findstr :3306

# Matar processo na porta
taskkill /PID <PID> /F

# Ou alterar porta no docker-compose.yml
ports:
  - "8090:80"  # Mudar de 8080 para 8090
```

---

## 📂 Estrutura de Pastas do CakePHP

```
cake_php/
├── bin/                    # Executáveis (cake, cake.php)
├── config/                 # Arquivos de configuração
│   ├── app.php            # Configurações principais
│   ├── app_local.php      # Configurações locais (DB)
│   ├── bootstrap.php      # Bootstrap da aplicação
│   └── routes.php         # Definição de rotas
├── logs/                   # Logs da aplicação
├── plugins/                # Plugins instalados
├── resources/              # Assets não compilados
├── src/                    # Código fonte da aplicação
│   ├── Controller/        # Controllers
│   ├── Model/             # Models (Table e Entity)
│   ├── View/              # Views e Helpers
│   └── Application.php    # Classe principal
├── templates/              # Templates (views)
│   ├── layout/            # Layouts
│   ├── element/           # Elementos reutilizáveis
│   └── Pages/             # Páginas estáticas
├── tests/                  # Testes automatizados
├── tmp/                    # Arquivos temporários e cache
│   ├── cache/             # Cache da aplicação
│   └── sessions/          # Sessões
├── vendor/                 # Dependências do Composer
├── webroot/                # Document root público
│   ├── css/               # Arquivos CSS
│   ├── js/                # Arquivos JavaScript
│   ├── img/               # Imagens
│   └── index.php          # Front controller
├── .gitignore
├── composer.json           # Dependências PHP
└── composer.lock
```

---

## 📚 Recursos Adicionais

### Documentação Oficial

- **CakePHP:** https://book.cakephp.org/5/pt/
- **Docker:** https://docs.docker.com/
- **NGINX:** https://nginx.org/en/docs/
- **PHP-FPM:** https://www.php.net/manual/pt_BR/install.fpm.php

### Tutoriais Recomendados

- [CakePHP 5 - Guia Completo](https://book.cakephp.org/5/pt/intro.html)
- [Docker Compose - Tutorial](https://docs.docker.com/compose/gettingstarted/)
- [NGINX + PHP-FPM Setup](https://www.nginx.com/resources/wiki/start/topics/examples/phpfcgi/)

---

## ✅ Checklist de Implementação

Use esta lista para acompanhar o progresso:

- [ ] **Fase 1:** Backup realizado
- [ ] **Fase 2:** Estrutura CakePHP reorganizada
- [ ] **Fase 3:** `docker-compose.yml` criado
- [ ] **Fase 4:** `nginx/default.conf` configurado
- [ ] **Fase 5:** `app_local.php` atualizado
- [ ] **Fase 6:** Containers Docker iniciados
- [ ] **Fase 7:** Dependências PHP instaladas
- [ ] **Fase 8:** Testes validados
- [ ] ✅ Aplicação acessível em http://localhost:8080
- [ ] ✅ PhpMyAdmin acessível em http://localhost:8081
- [ ] ✅ Banco de dados conectado
- [ ] ✅ Permissões ajustadas (tmp, logs, webroot)

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

---

## 📧 Suporte

Para dúvidas ou problemas:

- Verifique a seção **Solução de Problemas**
- Consulte os logs dos containers
- Acesse a documentação oficial do CakePHP

---

**Desenvolvido com ❤️ usando CakePHP, Docker e NGINX**

**Última atualização:** Janeiro 2026