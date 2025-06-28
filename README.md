#!/bin/bash
# Script para restaurar servidor GLPI a partir de backup .tar.gz no Ubuntu Server
# Use com cuidado e ajuste paths, nomes e senhas conforme seu ambiente.

set -e  # Para o script se ocorrer algum erro

echo "=== Iniciando restauração do GLPI ==="

# 1. Montar HD externo (ajuste o dispositivo /dev/sdb2 conforme seu HD)
echo "Montando HD externo..."
sudo mount /dev/sdb2 /mnt/hdexterno

# Validar montagem
df -h | grep /mnt/hdexterno
ls -lh /mnt/hdexterno

# 2. Copiar backup do HD externo para /tmp no servidor
echo "Copiando backup para /tmp..."
sudo cp /mnt/hdexterno/backup_completo_glpi_*.tar.gz /tmp/

# Confirmar arquivo copiado
ls -lh /tmp/backup_completo_glpi_*.tar.gz

# 3. Criar pasta para extrair o backup
echo "Criando pasta para extração do backup..."
mkdir -p /tmp/restore_glpi

# 4. Extrair backup completo
echo "Extraindo backup..."
sudo tar -xzvf /tmp/backup_completo_glpi_*.tar.gz -C /tmp/restore_glpi

# Validar extração
ls -lh /tmp/restore_glpi/backup-glpi

# 5. Instalar Apache, MariaDB, PHP e módulos necessários
echo "Instalando dependências..."
sudo apt update
sudo apt install apache2 mariadb-server php php-mysql libapache2-mod-php -y
sudo apt install php-curl php-xml php-mbstring php-zip php-gd php-bcmath php-intl -y

# Validar módulos PHP instalados
php -m | grep -Ei 'curl|xml|mbstring|zip|gd|bcmath|intl|mysql'

# Verificar status dos serviços
sudo systemctl status apache2 --no-pager
sudo systemctl status mysql --no-pager

# 6. Restaurar banco de dados
echo "Restaurando banco de dados GLPI..."

sudo mysql -u root -p <<EOF
CREATE DATABASE IF NOT EXISTS glpi CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER IF NOT EXISTS 'glpiuser'@'localhost' IDENTIFIED BY 'fc14f81t';
GRANT ALL PRIVILEGES ON glpi.* TO 'glpiuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
EOF

# Importar dump SQL
mysql -u root -p glpi < /tmp/restore_glpi/backup-glpi/banco_glpi_*.sql

# Verificar tabelas importadas
mysql -u glpiuser -pfc14f81t -e "USE glpi; SHOW TABLES;"

# 7. Extrair arquivos do GLPI para pasta web
echo "Extraindo arquivos GLPI para /var/www/html/glpi..."
sudo tar -xzvf /tmp/restore_glpi/backup-glpi/glpi_files_*.tar.gz -C /

# Confirmar arquivos no local
ls -lh /var/www/html/glpi

# 8. Ajustar permissões
echo "Ajustando permissões..."
sudo chown -R www-data:www-data /var/www/html/glpi
sudo chmod -R 755 /var/www/html/glpi
ls -ld /var/www/html/glpi

# 9. Limpar cache do GLPI
echo "Limpando cache do GLPI..."
sudo rm -rf /var/www/html/glpi/files/_cache/*

# 10. Atualizar config do banco no GLPI
echo "Atualize o arquivo /var/www/html/glpi/config/config_db.php para refletir o usuário e senha do banco."

# Exemplo de conteúdo para config_db.php
echo '<?php
class DB extends DBmysql {
    public $dbhost = "localhost";
    public $dbuser = "glpiuser";
    public $dbpassword = "fc14f81t";
    public $dbdefault = "glpi";
}
?>' | sudo tee /var/www/html/glpi/config/config_db.php > /dev/null

echo "Restauracao concluída! Acesse o GLPI pelo navegador: http://IP_DO_SERVIDOR/glpi"
echo "Usuário padrão: glpi / Senha padrão: glpi"
