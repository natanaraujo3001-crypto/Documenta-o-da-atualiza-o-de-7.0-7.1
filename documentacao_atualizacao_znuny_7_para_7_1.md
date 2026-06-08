# Documentação de atualização do Znuny 7.0 para 7.1

Baseada no procedimento executado no container, nos logs fornecidos e na documentação oficial de atualização do Znuny 7.1.

**Data de geração:** 08/06/2026  
**Tipo de atualização:** por código-fonte (`tar.gz`), não RPM.

> **Atenção:** este roteiro foi montado para o ambiente observado no chat: instalação dentro de container Docker/OrbStack, usuário da aplicação `otrs`, grupo web `www-data`, banco MariaDB local e symlink `/opt/znuny`. Em ambientes diferentes, ajuste usuário, grupo, caminhos e serviços antes de executar.

## 1. Ambiente usado como referência

| Item | Valor observado |
|---|---|
| Versão origem | Znuny 7.0.19 em `/opt/znuny-7.0.19` |
| Versão destino | Znuny 7.1.7 em `/opt/znuny-7.1.7` |
| Symlink ativo ao final | `/opt/znuny -> /opt/znuny-7.1.7` |
| Usuário da aplicação | `otrs` |
| Grupo web | `www-data` |
| Banco de dados | MariaDB, database `otrs`, host `127.0.0.1` |
| Container observado | `4b868009b364` |
| Web server | Apache 2.4 em Ubuntu |

## 2. Objetivo

Atualizar uma instalação Znuny 7.0.x para Znuny 7.1.x preservando configuração, banco de dados, dados da aplicação, cron jobs e arquivos customizados.

A atualização por código-fonte segue esta lógica:

1. Fazer backup completo.
2. Copiar o backup para fora do container.
3. Baixar e extrair a nova versão em `/opt`.
4. Copiar configuração e arquivos customizados da instalação antiga.
5. Trocar o symlink `/opt/znuny` para a nova versão.
6. Ajustar permissões.
7. Executar `scripts/MigrateToZnuny7_1.pl`.
8. Reconstruir configuração, limpar cache e reiniciar serviços.

## 3. Pré-requisitos e cuidados

Antes de iniciar:

- Agendar janela de manutenção.
- Confirmar que todos os add-ons/pacotes usados existem para Znuny 7.1.
- Garantir que o MariaDB esteja ativo.
- Fazer backup completo do sistema.
- Copiar o backup para fora do container.
- Não abrir o sistema para usuários enquanto a migração não terminar com `Migration completed!`.

> **Importante:** não use comandos exploratórios perigosos vistos no histórico, como `DROP DATABASE`. O roteiro abaixo contém somente a sequência limpa e recomendada.

Comandos úteis de conferência:

```bash
service mariadb status
service apache2 status
service cron status

ls -l /opt/znuny
find / -path "*/scripts/backup.pl" -type f 2>/dev/null
```

## 4. Backup completo antes da atualização

O script `scripts/backup.pl` faz backup do Znuny. No ambiente analisado, ele só concluiu depois que o MariaDB foi iniciado.

### 4.1. Criar diretório de backup

```bash
mkdir -p /backup/znuny_7.1
chown -R otrs:www-data /backup/znuny_7.1
chmod 775 /backup/znuny_7.1
```

### 4.2. Iniciar MariaDB

```bash
service mariadb start
```

### 4.3. Executar o backup como usuário da aplicação

```bash
cd /opt/znuny
su -s /bin/bash otrs -c "scripts/backup.pl -d /backup/znuny_7.1 -t fullbackup -c gzip"
```

Se o MariaDB estiver parado, pode ocorrer erro semelhante a:

```text
mysqldump: Got error: 2002: "Can't connect to server on '127.0.0.1'"
```

Correção:

```bash
service mariadb start
cd /opt/znuny
su -s /bin/bash otrs -c "scripts/backup.pl -d /backup/znuny_7.1 -t fullbackup -c gzip"
```

### 4.4. Validar o backup

```bash
ls -lh /backup/znuny_7.1
ls -lh /backup/znuny_7.1/<DATA_HORA_DO_BACKUP>

gzip -t /backup/znuny_7.1/<DATA_HORA_DO_BACKUP>/*.gz
echo $?
```

Se `echo $?` retornar `0`, os arquivos `.gz` testados estão íntegros.

### 4.5. Copiar o backup para fora do container

Execute no terminal do host, fora do container:

```bash
docker cp 4b868009b364:/backup/znuny_7.1/<DATA_HORA_DO_BACKUP> ./znuny_backup_7.1
ls -lh ./znuny_backup_7.1
```

## 5. Baixar e extrair o Znuny 7.1

No container, execute:

```bash
cd /opt
wget https://download.znuny.org/releases/znuny-latest-7.1.tar.gz
tar xfz znuny-latest-7.1.tar.gz
ls -ld /opt/znuny-7.1*
```

No caso executado, o diretório extraído foi:

```text
/opt/znuny-7.1.7
```

## 6. Ajustar permissões da nova instalação

No ambiente analisado, o comando sem parâmetros falhou:

```text
ERROR: --znuny-user is missing or invalid.
```

Como o usuário da aplicação é `otrs`, o comando correto foi:

```bash
/opt/znuny-7.1.7/bin/znuny.SetPermissions.pl --znuny-user otrs --web-group www-data
```

> **Nota:** a opção `--znuny-home` não foi reconhecida nesse ambiente. Use apenas `--znuny-user` e `--web-group`.

## 7. Copiar configuração e arquivos customizados

Copie o `Config.pm` da instalação atual para a nova versão:

```bash
cp -a /opt/znuny/Kernel/Config.pm /opt/znuny-7.1.7/Kernel/
```

Copie artigos/anexos somente se o diretório existir:

```bash
mkdir -p /opt/znuny-7.1.7/var/article
if [ -d /opt/znuny/var/article ]; then
    cp -a /opt/znuny/var/article/. /opt/znuny-7.1.7/var/article/
fi
```

> **Importante:** use `cp -a`, não `mv`, durante o upgrade. Assim a versão antiga permanece intacta caso precise reverter.

Copie dotfiles customizados:

```bash
for f in $(find -L /opt/znuny -maxdepth 1 -type f -name ".*" -not -name "*.dist"); do
    cp -av "$f" /opt/znuny-7.1.7/
done
```

Copie cron jobs customizados/modificados:

```bash
mkdir -p /opt/znuny-7.1.7/var/cron
for f in $(find -L /opt/znuny/var/cron -maxdepth 1 -type f -name "*" -not -name "*.dist" 2>/dev/null); do
    cp -av "$f" /opt/znuny-7.1.7/var/cron/
done
```

## 8. Atualizar o symlink `/opt/znuny`

```bash
ln -snf /opt/znuny-7.1.7 /opt/znuny
ls -l /opt/znuny
```

Resultado esperado:

```text
/opt/znuny -> /opt/znuny-7.1.7
```

Depois, reaplique permissões:

```bash
/opt/znuny/bin/znuny.SetPermissions.pl --znuny-user otrs --web-group www-data
```

## 9. Conferir módulos Perl

```bash
/opt/znuny/bin/znuny.CheckModules.pl --all
```

No caso executado, alguns módulos opcionais apareceram como não instalados, mas não impediram a migração. O ponto crítico é não haver módulo obrigatório ausente.

## 10. Executar a migração para 7.1

```bash
service mariadb start
cd /opt/znuny
su -s /bin/bash otrs -c "scripts/MigrateToZnuny7_1.pl --verbose"
```

Responda `y` quando o script perguntar:

```text
Did you backup the database? [Y]es/[N]o:
```

Responda `y` também quando perguntar:

```text
Should the SysConfig be migrated? [Y]es/[N]o:
```

O resultado correto é:

```text
Migration completed!
```

## 11. Problemas encontrados e correções

### 11.1. Erro: arquivos ZZZ ausentes

Sintoma:

```text
Can't locate Kernel/Config/Files/ZZZAAuto.pm
Can't locate Kernel/Config/Files/ZZZACL.pm
Can't locate Kernel/Config/Files/ZZZProcessManagement.pm
```

Causa: os arquivos de SysConfig gerados não foram copiados para a nova versão.

Correção:

```bash
mkdir -p /opt/znuny-7.1.7/Kernel/Config/Files
cp -av /opt/znuny-7.0.19/Kernel/Config/Files/ZZZ*.pm /opt/znuny-7.1.7/Kernel/Config/Files/
/opt/znuny/bin/znuny.SetPermissions.pl --znuny-user otrs --web-group www-data
```

### 11.2. Erro: `character_set_database needs to be 'utf8'`

Sintoma inicial:

```text
Step 8 of 22: Check database charset ...
The setting character_set_client is: utf8mb4.
Error: The setting character_set_database needs to be 'utf8'.
```

Foi tentado:

```bash
mariadb -e "ALTER DATABASE otrs CHARACTER SET utf8 COLLATE utf8_general_ci;"
mariadb -e "USE otrs; SHOW VARIABLES LIKE 'character_set_database'; SHOW VARIABLES LIKE 'collation_database';"
```

O banco ficou como `utf8mb3`, mas a migração ainda falhou. A causa real era que o `Kernel/Config.pm` ainda continha:

```perl
$Self->{Home} = '/opt/otrs';
```

Esse valor fazia o Znuny usar caminhos antigos, como `/opt/otrs`, durante a migração. O correto para este ambiente é:

```perl
$Self->{Home} = '/opt/znuny';
```

Correção aplicada:

```bash
cp -a /opt/znuny/Kernel/Config.pm /opt/znuny/Kernel/Config.pm.bak-home-$(date +%F-%H%M)
sed -i "s|\$Self->{Home} = '/opt/otrs';|\$Self->{Home} = '/opt/znuny';|" /opt/znuny/Kernel/Config.pm
grep -n "Home" /opt/znuny/Kernel/Config.pm
```

Depois:

```bash
cd /opt/znuny
rm -rf var/tmp/*
rm -rf var/sessions/*
/opt/znuny/bin/znuny.SetPermissions.pl --znuny-user otrs --web-group www-data
su -s /bin/bash otrs -c "scripts/MigrateToZnuny7_1.pl --verbose"
```

Após essa correção, a migração passou pelo charset com:

```text
The setting character_set_client is: utf8mb4. The setting character_set_database is: utf8mb4. No tables found with invalid charset.
```

E terminou com:

```text
Migration completed!
```

### 11.3. Internal Server Error 500 após trocar o symlink

Sintoma no navegador:

```text
Internal Server Error
Apache/2.4.66 (Ubuntu)
```

Causa provável: aplicação já apontava para a versão nova, mas a migração ainda não tinha terminado.

Correção: resolver os erros acima, repetir a migração até `Migration completed!`, reconstruir configuração, limpar cache e reiniciar serviços.

## 12. Pós-migração

Depois que a migração termina com sucesso:

```bash
cd /opt/znuny
su -s /bin/bash otrs -c "bin/znuny.Console.pl Maint::Config::Rebuild"
su -s /bin/bash otrs -c "bin/znuny.Console.pl Maint::Cache::Delete"
su -s /bin/bash otrs -c "bin/znuny.Console.pl Admin::Package::ReinstallAll"
```

Reinicie os serviços:

```bash
apache2ctl configtest
service apache2 restart
service cron start
su -s /bin/bash otrs -c "bin/znuny.Daemon.pl start"
su -s /bin/bash otrs -c "bin/znuny.Daemon.pl status"
```

## 13. Validação final

Validar:

- Interface web abre sem erro 500.
- Login de agente/admin funciona.
- Tickets antigos abrem corretamente.
- Artigos/anexos estão acessíveis.
- Criação ou atualização de ticket funciona.
- Serviços de e-mail funcionam, se usados.
- Logs não mostram erro crítico após reiniciar Apache.

Comandos úteis:

```bash
tail -n 100 /var/log/apache2/error.log
tail -n 100 /opt/znuny/var/log/znuny.log 2>/dev/null
```

## 14. Sequência limpa de comandos de referência

Ajuste `/opt/znuny-7.1.7` caso a versão extraída tenha outro número.

```bash
# Backup
mkdir -p /backup/znuny_7.1
chown -R otrs:www-data /backup/znuny_7.1
chmod 775 /backup/znuny_7.1
service mariadb start
cd /opt/znuny
su -s /bin/bash otrs -c "scripts/backup.pl -d /backup/znuny_7.1 -t fullbackup -c gzip"

# Baixar nova versão
cd /opt
wget https://download.znuny.org/releases/znuny-latest-7.1.tar.gz
tar xfz znuny-latest-7.1.tar.gz

# Permissões iniciais
/opt/znuny-7.1.7/bin/znuny.SetPermissions.pl --znuny-user otrs --web-group www-data

# Restaurar arquivos
cp -a /opt/znuny/Kernel/Config.pm /opt/znuny-7.1.7/Kernel/
mkdir -p /opt/znuny-7.1.7/var/article
if [ -d /opt/znuny/var/article ]; then
    cp -a /opt/znuny/var/article/. /opt/znuny-7.1.7/var/article/
fi

for f in $(find -L /opt/znuny -maxdepth 1 -type f -name ".*" -not -name "*.dist"); do
    cp -av "$f" /opt/znuny-7.1.7/
done

mkdir -p /opt/znuny-7.1.7/var/cron
for f in $(find -L /opt/znuny/var/cron -maxdepth 1 -type f -name "*" -not -name "*.dist" 2>/dev/null); do
    cp -av "$f" /opt/znuny-7.1.7/var/cron/
done

# Trocar symlink
ln -snf /opt/znuny-7.1.7 /opt/znuny
/opt/znuny/bin/znuny.SetPermissions.pl --znuny-user otrs --web-group www-data
/opt/znuny/bin/znuny.CheckModules.pl --all

# Correções específicas deste ambiente
mkdir -p /opt/znuny-7.1.7/Kernel/Config/Files
cp -av /opt/znuny-7.0.19/Kernel/Config/Files/ZZZ*.pm /opt/znuny-7.1.7/Kernel/Config/Files/

cp -a /opt/znuny/Kernel/Config.pm /opt/znuny/Kernel/Config.pm.bak-home-$(date +%F-%H%M)
sed -i "s|\$Self->{Home} = '/opt/otrs';|\$Self->{Home} = '/opt/znuny';|" /opt/znuny/Kernel/Config.pm

cd /opt/znuny
rm -rf var/tmp/*
rm -rf var/sessions/*
/opt/znuny/bin/znuny.SetPermissions.pl --znuny-user otrs --web-group www-data

# Migração
service mariadb start
cd /opt/znuny
su -s /bin/bash otrs -c "scripts/MigrateToZnuny7_1.pl --verbose"

# Pós-migração
su -s /bin/bash otrs -c "bin/znuny.Console.pl Maint::Config::Rebuild"
su -s /bin/bash otrs -c "bin/znuny.Console.pl Maint::Cache::Delete"
su -s /bin/bash otrs -c "bin/znuny.Console.pl Admin::Package::ReinstallAll"

apache2ctl configtest
service apache2 restart
service cron start
su -s /bin/bash otrs -c "bin/znuny.Daemon.pl start"
su -s /bin/bash otrs -c "bin/znuny.Daemon.pl status"
```

## 15. Rollback básico em caso de falha

Se a migração não terminar e for necessário voltar, faça rollback somente com backup válido.

- Parar Apache/cron/daemon.
- Voltar o symlink para a versão antiga.
- Restaurar o banco usando o `DatabaseBackup.sql.gz` do backup.
- Reaplicar permissões.
- Reiniciar serviços.

Exemplo conceitual:

```bash
service apache2 stop
service cron stop
su -s /bin/bash otrs -c "bin/znuny.Daemon.pl stop" 2>/dev/null

ln -snf /opt/znuny-7.0.19 /opt/znuny
/opt/znuny/bin/znuny.SetPermissions.pl --znuny-user otrs --web-group www-data

# Restaurar banco somente se você tiver certeza do arquivo correto:
# gzip -dc /backup/znuny_7.1/<DATA_HORA_DO_BACKUP>/DatabaseBackup.sql.gz | mariadb otrs

service mariadb restart
service apache2 restart
service cron start
```

> **Atenção:** depois que a migração altera o banco, voltar apenas o symlink não basta. É necessário restaurar o banco do backup correspondente.

## 16. Fontes e evidências usadas

- Documentação oficial do Znuny 7.1: atualização por source, backup obrigatório, verificação de `$Self->{Home}`, symlink e execução do script `MigrateToZnuny7_1.pl`.
- Logs fornecidos no arquivo anexado: backup, erros de MariaDB, ausência de arquivos `ZZZ*.pm`, erro de charset, correção de `$Self->{Home}` e conclusão com `Migration completed!`.
- Histórico técnico do chat: comandos executados, correções aplicadas e validações feitas durante o procedimento.
