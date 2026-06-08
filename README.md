# Documentação da Atualização do Znuny 7.0 para 7.1

Este repositório contém a documentação do processo de atualização do **Znuny 7.0.19 para o Znuny 7.1.7**, realizado em ambiente baseado em container.

O objetivo deste material é registrar o passo a passo executado, os erros encontrados durante a atualização e as correções aplicadas até a conclusão da migração.

## Conteúdo da documentação

A documentação cobre os seguintes pontos:

* criação de backup completo do Znuny;
* validação dos arquivos de backup;
* cópia do backup para fora do container;
* download e extração do pacote Znuny 7.1;
* ajuste de permissões da nova instalação;
* cópia dos arquivos de configuração da versão anterior;
* atualização do symlink `/opt/znuny`;
* verificação dos módulos Perl necessários;
* execução da migração para Znuny 7.1;
* correção de erros encontrados durante a migração;
* limpeza de cache;
* reconstrução da configuração;
* reinício dos serviços;
* validação final do ambiente.

## Ambiente utilizado

A atualização foi realizada em um ambiente com as seguintes características:

* Znuny origem: `7.0.19`
* Znuny destino: `7.1.7`
* Banco de dados: MariaDB
* Servidor web: Apache
* Usuário da aplicação: `otrs`
* Grupo web: `www-data`
* Diretório principal da aplicação: `/opt/znuny`
* Diretório de backup utilizado: `/backup/znuny_7.1`

## Problemas encontrados

Durante o processo, alguns problemas foram identificados e corrigidos:

### MariaDB parado durante o backup

O backup falhou inicialmente porque o MariaDB não estava em execução.

Correção aplicada:

```bash
service mariadb start
```

Após iniciar o banco, o backup foi executado novamente com sucesso.

### Falha no `znuny.SetPermissions.pl`

O comando inicial falhou porque o script exigia a indicação explícita do usuário do Znuny.

Correção aplicada:

```bash
/opt/znuny/bin/znuny.SetPermissions.pl --znuny-user otrs --web-group www-data
```

### Diretório `/var/article` inexistente

Durante a cópia dos artigos/anexos, o diretório `/opt/znuny/var/article` não existia.

Foi utilizada uma cópia condicional para evitar erro:

```bash
mkdir -p /opt/znuny-7.1.7/var/article

if [ -d /opt/znuny/var/article ]; then
    cp -a /opt/znuny/var/article/. /opt/znuny-7.1.7/var/article/
fi
```

### Arquivos `ZZZ*.pm` ausentes

A migração apresentou erro informando ausência dos arquivos:

* `ZZZAAuto.pm`
* `ZZZACL.pm`
* `ZZZProcessManagement.pm`

Correção aplicada:

```bash
mkdir -p /opt/znuny-7.1.7/Kernel/Config/Files
cp -av /opt/znuny-7.0.19/Kernel/Config/Files/ZZZ*.pm /opt/znuny-7.1.7/Kernel/Config/Files/
```

### Configuração `Home` apontando para `/opt/otrs`

O arquivo `Kernel/Config.pm` ainda possuía:

```perl
$Self->{Home} = '/opt/otrs';
```

Isso causava comportamento incorreto durante a migração.

Correção aplicada:

```bash
sed -i "s|\$Self->{Home} = '/opt/otrs';|\$Self->{Home} = '/opt/znuny';|" /opt/znuny/Kernel/Config.pm
```

Resultado esperado:

```perl
$Self->{Home} = '/opt/znuny';
```

### Erro de charset no banco

Durante a migração, ocorreu erro relacionado ao charset do banco de dados.

Foi necessário ajustar o charset do banco `otrs` e executar novamente a migração:

```bash
mariadb -e "ALTER DATABASE otrs CHARACTER SET utf8 COLLATE utf8_general_ci;"
```

Depois da correção do `Home`, a migração conseguiu validar corretamente o banco em `utf8mb4`.

## Resultado final

Após as correções, o script de migração foi executado com sucesso e retornou:

```text
Migration completed!
```

Isso indica que a atualização para o Znuny 7.1 foi concluída corretamente.

## Comandos principais usados

### Backup

```bash
service mariadb start

mkdir -p /backup/znuny_7.1
chown -R otrs:www-data /backup/znuny_7.1
chmod 775 /backup/znuny_7.1

cd /opt/znuny
su -s /bin/bash otrs -c "scripts/backup.pl -d /backup/znuny_7.1 -t fullbackup -c gzip"
```

### Download e extração do Znuny 7.1

```bash
cd /opt
wget https://download.znuny.org/releases/znuny-latest-7.1.tar.gz
tar xfz znuny-latest-7.1.tar.gz
```

### Ajuste de permissões

```bash
/opt/znuny-7.1.7/bin/znuny.SetPermissions.pl --znuny-user otrs --web-group www-data
```

### Cópia de configurações

```bash
cp -a /opt/znuny/Kernel/Config.pm /opt/znuny-7.1.7/Kernel/

mkdir -p /opt/znuny-7.1.7/var/article

if [ -d /opt/znuny/var/article ]; then
    cp -a /opt/znuny/var/article/. /opt/znuny-7.1.7/var/article/
fi

mkdir -p /opt/znuny-7.1.7/var/cron

for f in $(find -L /opt/znuny/var/cron -maxdepth 1 -type f -name "*" -not -name "*.dist" 2>/dev/null); do
    cp -av "$f" /opt/znuny-7.1.7/var/cron/
done
```

### Atualização do symlink

```bash
ln -snf /opt/znuny-7.1.7 /opt/znuny
```

### Correção do `Home`

```bash
cp -a /opt/znuny/Kernel/Config.pm /opt/znuny/Kernel/Config.pm.bak-home-$(date +%F-%H%M)

sed -i "s|\$Self->{Home} = '/opt/otrs';|\$Self->{Home} = '/opt/znuny';|" /opt/znuny/Kernel/Config.pm
```

### Migração

```bash
cd /opt/znuny
su -s /bin/bash otrs -c "scripts/MigrateToZnuny7_1.pl --verbose"
```

### Pós-migração

```bash
su -s /bin/bash otrs -c "bin/znuny.Console.pl Maint::Config::Rebuild"
su -s /bin/bash otrs -c "bin/znuny.Console.pl Maint::Cache::Delete"
su -s /bin/bash otrs -c "bin/znuny.Console.pl Admin::Package::ReinstallAll"
```

### Reinício dos serviços

```bash
service apache2 restart
service cron start
su -s /bin/bash otrs -c "bin/znuny.Daemon.pl start"
```

## Observações

Antes de executar qualquer atualização em ambiente de produção, é obrigatório realizar backup completo do banco de dados e dos arquivos da aplicação.

Também é recomendado copiar o backup para fora do container ou servidor antes de iniciar a migração.

Este repositório serve como registro técnico do procedimento realizado e pode ser usado como referência para futuras atualizações do Znuny.
