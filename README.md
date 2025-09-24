# Wordpress-aws
Este README descreve o processo para implantar o WordPress na AWS com VPC customizada, LoadBalance, Auto-Scaling, EFS e RDS. 

requisitos: conta AWS com permissão para EC2, RDS, EFS, ELB, IAM e VPC.  

1 — Visão geral 
- VPC 
- 2 subnets públicas   
- 4 subnets privadas 
- EFS 
- RDS MySQL em subnets privadas  

2 — Passo a passo resumido

2.1 Criar a VPC 
1. Criar VPC `10.0.0.0/16`.  
2. Criar duas subnets públicas (ex.: `10.0.1.0/24`, `10.0.2.0/24`) em AZs distintas.  
3. Criar quatro subnets privadas (ex.: `10.0.3.0/24`…`10.0.6.0/24`).  
4. Criar Internet Gateway  e associar a tabela de rota pública (`0.0.0.0/0 → IGW`).  
5. Criar NAT Gateway nas subnets públicas e adicionar rotas `0.0.0.0/0 → NAT` nas tabelas privadas.

2.2 Criar Security Groups 
- SG-ALB: Entrada HTTP 80 (0.0.0.0/0), HTTPS 443 (0.0.0.0/0)  
- SG-EC2: Entrada HTTP 80 (origem = SG-ALB), SSH 22 (se necessário, origem = seu IP), NFS 2049 (origem = SG-EFS). Saida: Tudo
- SG-RDS: Entrada MySQL 3306 (origem = SG-EC2)  
- SG-EFS: Entrada NFS 2049 (origem = SG-EC2)

2.3 Criar RDS 
- Console RDS → Create database (Standard create)  
- Engine: MySQL   
- Instance class: db.t3.micro   
- DB name: `wordpress`   
- DB user: `admin` / senha   
- Network: selecione a VPC e crie um DB subnet group usando as subnets privadas  
- SG: RDS

2.4 Criar EFS
- Console EFS → Create file system → selecionar a VPC  
- Criar mount targets em subnet  
- SG: EFS

2.5 Preparar IAM Role 
- Criar Regra para EC2 com `AmazonSSMManagedInstanceCore`  e  `secretsmanager:GetSecretValue`

2.6 Criar o Launch Template 
- EC2 → Launch Templates → Create  
- AMI: Amazon Linux 2023  
- Instancia: `t3.micro`  
- SG: `SG-EC2`
- Selecionar rede publica
- Habilitar Ip publico automatico
- IAM : regra criada  
- SCRIPT DO USERDATA:
 #!/bin/bash
exec > >(tee /var/log/user-data.log|logger -t user-data -s 2>/dev/console) 2>&1
set -e
REGION="us-east-1"
EFS_ID="fs-0124b0bdc3b02c139"
RDS_ENDPOINT="meu-banco.cmbmwcqseiw5.us-east-1.rds.amazonaws.com"
DB_USER="admin"
DB_PASS="73540785"
DB_NAME="wordpress"
echo "[INFO] Atualizando pacotes e instalando dependências..."
dnf update -y
dnf install -y httpd php php-mysqlnd php-mbstring php-xml amazon-efs-utils curl tar unzip rsync
systemctl enable httpd
systemctl start httpd
mkdir -p /var/www/html
chown -R apache:apache /var/www/html
MOUNT_POINT="/var/www/html/wp-content/uploads"
mkdir -p ${MOUNT_POINT}
if ! grep -q "${EFS_ID}" /etc/fstab; then
  echo "${EFS_ID}:/ ${MOUNT_POINT} efs _netdev,tls 0 0" >> /etc/fstab
fi
mount -a || true
if [ ! -f /var/www/html/wp-config.php ]; then
  echo "[INFO] Baixando WordPress..."
  cd /tmp
  curl -O https://wordpress.org/latest.tar.gz
  tar xzf latest.tar.gz
  rsync -a wordpress/ /var/www/html/
  cp /var/www/html/wp-config-sample.php /var/www/html/wp-config.php
  echo "[INFO] Configurando wp-config.php..."
  sed -i "s/database_name_here/${DB_NAME}/" /var/www/html/wp-config.php
  sed -i "s/username_here/${DB_USER}/" /var/www/html/wp-config.php
  sed -i "s/password_here/${DB_PASS}/" /var/www/html/wp-config.php
  sed -i "s/localhost/${RDS_ENDPOINT}/" /var/www/html/wp-config.php
  SALTS=$(curl -s https://api.wordpress.org/secret-key/1.1/salt/)
  printf "\n%s\n" "$SALTS" >> /var/www/html/wp-config.php
  chown -R apache:apache /var/www/html
fi
cat >/var/www/html/healthz.php <<'PHP'
<?php http_response_code(200); echo "OK"; ?>
PHP
chown apache:apache /var/www/html/healthz.php
systemctl restart httpd
echo "[INFO] Setup concluído com sucesso."

2.7 Criar Target Group 
- EC2 → Target Groups → Create  
- Tipo: `instance`, Protocol: HTTP, Port: 80, VPC: sua VPC  
- Health check path: `/healthz.php`  
- Name: `wp-tg`

2.8 Criar Application Load Balancer 
- EC2 → Load Balancers → Create → Application Load Balancer  
- Scheme: `internet-facing`  
- Subnets: as públicas  
- Security group: `SG-ALB`  
- Listener HTTP 80 → action: forward para `wp-tg` (peso 1)

2.9 Criar Auto Scaling Group 
- EC2 → Auto Scaling Groups → Create  
- Escolha o Launch Template criado  
- VPC: sua VPC  
- Subnets: selecione as subnets privadas  
- Load balancing: associar ao Target Group `wp-tg`  
- Group size: Min = 1, Desired = 1, Max = 2 

# 3 — Testes e validação rápidos
- Acesse o DNS do ALB → deve abrir WordPress.  
- Verifique targets health (status `healthy`).  
- Em uma instância do ASG:  
  - `curl -I http://localhost/healthz.php` → deve retornar `200 OK`  
  - `mount | grep efs` → confirmar EFS montado  
  - `mysql -h RDS_ENDPOINT -u admin -p` → testar conexão ao RDS  

