---
layout: post
title: "elasticsearch: configurando a segurança"
description: "tutorial e exemplo de como configurar a segurança e autenticação no elasticsearch 8"
date: 2022-10-26 20:30:00 -0300
lang: pt-br
ref: "p0008"
categories: [ posts, pt-br ]
tags: [ elasticsearch, elastic ]
cover: /img/posts/covers/0008.png
---
Um dos pontos mais importantes na hora de implementar um cluster Elasticsearch são as configurações relacionadas a segurança, tanto a autenticação de usuários, quanto a forma como os nós se comunicam entre si e com os clientes.

Durante um bom tempo, a autenticação no Elasticsearch não era algo nativo, era necessário utilizar um plugin pago da própria Elastic, o antigo _x-pack_, um plugin de terceiros, como o _Search Guard_, ou implementar um proxy com autenticação básica para receber os requests e repassar para o cluster Elasticsearch.

Expor um cluster na internet sem autenticação é extremamente arriscado e é apenas uma questão de tempo para que ocorra algum vazamento ou perda de dados.

A partir das versões `6.8` e `7.1` a Elastic [passou a incluir][security-free] a possibilidade de habilitar a autenticação básica de forma nativa e sem custos com a licença `Basic` e a partir da versão `8.0` as funcionalidades de segurança vem habilitadas por padrão.

### Camadas de segurança

A Elastic define basicamente 3 camadas de segurança que podem ser aplicadas em um cluster.

**minimal**

Apenas a autenticação de usuário é habilitada, funcionada apenas para clusters que sejam `single-node`, não é recomendado para produção.

**basic security**

Além da autenticação de usuário, a comunicação entre os nós (_transport_) é criptografada utilizando TLS, pode ser utilizado em produção, mas os requests de clientes para o Elasticsearch usam `http` e portanto não são criptografados.

**basic security + https**

Utiliza TLS também para os requests feitos por clientes, é a configuração ideal para produção.

### Configurando a Segurança

A versão `8` do Elasticsearch facilita bastante o processo de implementação de um cluster com a segurança habilitada, mas em alguns cenários talvez seja necessário configurar manualmente os parâmetros de segurança.

Nesse exemplo iremos configurar manualmente a segurança em um cluster de **`3`** nós de Elasticsearch e **`1`** Kibana.

![es-security](/img/posts/0008/0008-01.jpg)

Nessa configuração os nós de Elasticsearch se comunicam utilizando TLS, essa comunicação entre os nós utiliza um protocolo chamado de _transport_ e a segurança é habilitada através dos parâmetros `xpack.security.transport.ssl.*`, além disso a comunicação dos clientes com qualquer nó do cluster utiliza requisições _REST_ via _HTTPS_, a segurança na comunicação com os clientes é habilitada através dos parâmetros `xpack.security.http.ssl.*`.

![es-security-xpack](/img/posts/0008/0008-02.jpg)

Para que essa comunicação seja possível precisamos de certificados para cada nó e uma autoridade certificadora para criar os certificados.

### Instalando o Elasticsearch

Nesse exemplo iremos utilizar servidores com _Rocky Linux 8_ e instalar o Elasticsearch a partir do pacote `rpm`, sem adicionar o repositório.

```bash
$ wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.4.3-x86_64.rpm
$ sudo rpm -ivh elasticsearch-8.4.3-x86_64.rpm
```

Durante o processo de instalação é executada uma autoconfiguração de segurança que cria certificados para o nó e o arquivo `elasticsearch.keystore` com as chaves para esses certificados, como queremos realizar uma configuração manual, iremos remover os certificados criados e também o arquivo `elasticsearch.keystore`.

```bash
$ sudo rm -rf /etc/elasticsearch/certs
$ sudo rm -f /etc/elasticsearch/elasticsearch.keystore
```

#### Bootstrap Checks

Durante a inicialização o elasticsearch vai realizar algumas validações para saber se o processo consegue alocar memória suficiente, para isso precisamos alterar tanto o arquivo `/etc/security/limits.conf` quanto adicionar uma configuração extra para o serviço `systemd` do elasticsearch.

No arquivo `/etc/security/limits.conf` adicionamos as linhas abaixo.

```
elasticsearch soft memlock unlimited
elasticsearch hard memlock unlimited
```

Para o serviço `systemd` precisamos antes criar o diretório `/etc/systemd/system/elasticsearch.service.d`

```bash
$ sudo mkdir /etc/systemd/system/elasticsearch.service.d
```

Na sequência adicionamos as linhas abaixo no arquivo `/etc/systemd/system/elasticsearch.service.d/override.conf`

```
[Service]
LimitMEMLOCK=infinity
```

Como alteramos um serviço `systemd`, precisamos executar um `daemon-reload`.

```bash
$ sudo systemctl daemon-reload
```

### Criando uma Autoridade Certificadora (CA)

Após a instalação do Elasticsearch, precisamos agora criar uma autoridade certificadora para gerar os certificados de cada nó, para isso iremos utilizar o comando `elasticsearch-certutil` no modo `ca` em um dos nós.

```bash
$ sudo /usr/share/elasticsearch/bin/elasticsearch-certutil ca --pem -pass
```

Esse comando irá gerar um arquivo `.zip` contendo a chave privada e o certificado, será solicitado uma senha pra chave privada e o nome do arquivo.

```
This tool assists you in the generation of X.509 certificates and certificate
signing requests for use with SSL/TLS in the Elastic stack.

The 'ca' mode generates a new 'certificate authority'
This will create a new X.509 certificate and private key that can be used
to sign certificate when running in 'cert' mode.

Use the 'ca-dn' option if you wish to configure the 'distinguished name'
of the certificate authority

By default the 'ca' mode produces a single PKCS#12 output file which holds:
    * The CA certificate
    * The CA's private key

If you elect to generate PEM format certificates (the -pem option), then the output will
be a zip file containing individual files for the CA certificate and private key

Enter password for CA Private key : 
Please enter the desired output file [elastic-stack-ca.zip]:
```

O arquivo `elastic-stack-ca.zip` será criado no caminho `/usr/share/elasticsearch` e iremos copiá-lo para o diretório `/etc/elasticsearch/certs`.

``` bash
$ sudo mkdir /etc/elasticsearch/certs
$ sudo cp /usr/share/elasticsearch/elastic-stack-ca.zip /etc/elasticsearch/certs
$ sudo unzip /etc/elasticsearch/certs/elastic-stack-ca.zip -d /etc/elasticsearch/certs/
```

Ao descompactarmos o arquivo `elastic-stack-ca.zip` teremos os arquivos `ca.crt` e `ca.key` 

```bash
/etc/elasticsearch
└── certs
    └── ca
        ├── ca.crt
        └── ca.key
```

### Criando os certificados para os nós<a name="nodecert"></a>

Podemos criar um certificado para as comunicações _transport_ e outro certificado diferente para as comunicações _https_, mas nesse exemplo iremos criar apenas um certificado por nó.

Para criar os certificados utilizaremos novamente o comando `elasticsearch-certutil`, dessa vez no modo `cert`.

```bash
$ sudo /usr/share/elasticsearch/bin/elasticsearch-certutil cert --silent --pem \
    --ca-key /etc/elasticsearch/certs/ca/ca.key \
    --ca-cert /etc/elasticsearch/certs/ca/ca.crt --ca-pass SENHA-CHAVE-CA \
    --in /tmp/instances.yml --out nodes.zip --pass PASSPHRASE-CERTIFICADOS
```

Onde os parâmetros são:

- `--silent`: exibe o mínimo possível como output
- `--pem`: gera certificados no formato PEM, ou seja, um arquivo `.crt` e um arquivo `.key`
- `--ca-key`: caminho para a chave privada da CA
- `--ca-cert`: caminho para o certificado da CA
- `--ca-pass`: senha da chave privada da CA
- `--in`: arquivo `.yml` com o nome, dns e ip de cada nó
- `--out`: nome do arquivo zip onde os certificados serão salvos
- `--pass`: _passphrase_ para os certificados dos nós

Antes de executar o comando precisamos criar o arquivo `instances.yml` no seguinte formato:

```yaml
instances:
  - name: "node-name"
    ip: ["ip"]
    dns: ["node-name", "node-name.domain.name]
```

Nesse arquivo colocamos todos os IPs que o nó possui e todas as entradas DNS que ele responde.

O arquivo `instances.yml` do nosso exemplo é o seguinte.

```yaml
instances:
  - name: "es01"
    ip: ["10.0.0.101"]
    dns: ["es01", "es01.lab", "es01.lab.local"]
  - name: "es02"
    ip: ["10.0.0.102"]
    dns: ["es02", "es02.lab", "es02.lab.local"]
  - name: "es03"
    ip: ["10.0.0.103"]
    dns: ["es03", "es03.lab", "es03.lab.local"]
```

Após a execução do comando o arquivo `nodes.zip` será criado no caminho `/usr/share/elasticsearch`, vamos copiá-lo para o caminho `/etc/elasticsearsh/certs`.

```bash
$ sudo cp /usr/share/elasticsearch/nodes.zip /etc/elasticsearch/certs/
$ sudo unzip /etc/elasticsearch/certs/nodes.zip -d /etc/elasticsearch/certs/
```

Será criado então um diretório para cada nó contendo o arquivo `.crt` e `.key` do nó.

```bash
Archive:  /etc/elasticsearch/certs/nodes.zip
   creating: /etc/elasticsearch/certs/es01/
  inflating: /etc/elasticsearch/certs/es01/es01.crt  
  inflating: /etc/elasticsearch/certs/es01/es01.key  
   creating: /etc/elasticsearch/certs/es02/
  inflating: /etc/elasticsearch/certs/es02/es02.crt  
  inflating: /etc/elasticsearch/certs/es02/es02.key  
   creating: /etc/elasticsearch/certs/es03/
  inflating: /etc/elasticsearch/certs/es03/es03.crt  
  inflating: /etc/elasticsearch/certs/es03/es03.key
```

Na sequência precisamos copiar o conteúdo do caminho `/etc/elasticsearch/certs` no nó onde criamos a CA e o certificado para os outros nós que farão parte do cluster.

### Configurando a segurança e iniciando o cluster

Antes de iniciarmos os nós do cluster, precisamos configurar as opções de segurança no arquivo `elasticsearch.yml`de cada nó, criar o arquivo `elasticsearch.keystore` e adicionar a passphrase da chave de cada nó.

Para criarmos o arquivo `elasticsearch.keystore` usamos o seguinte comando em cada um dos nós.

```bash
$ sudo /usr/share/elasticsearch/bin/elasticsearch-keystore create
$ sudo sh -c 'echo "passphrase" | /usr/share/elasticsearch/bin/elasticsearch-keystore add --stdin xpack.security.transport.ssl.secure_key_passphrase -f'
$ sudo sh -c 'echo "passphrase" | /usr/share/elasticsearch/bin/elasticsearch-keystore add --stdin xpack.security.http.ssl.secure_key_passphrase -f'
$ sudo chmod 0660 /etc/elasticsearch/elasticsearch.keystore
```

Com o keystore criado e configurado, usamos o seguinte arquivo `elasticsearch.yml` em cada um dos nós, alterando os valores específicos para cada nó.

Como exemplo para o nó `es01` temos a seguinte configuração.

```yaml
# es01
#
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
bootstrap.memory_lock: true

cluster.name: es
node.name: es01

network.host: ['10.0.0.101']
http.port: 9200

cluster.initial_master_nodes: ['es01', 'es02', 'es03']
discovery.seed_hosts: ['es01', 'es02', 'es03']

# security settings
xpack.security.enabled: true
xpack.security.autoconfiguration.enabled: false
# transport ssl
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.key: certs/es01/es01.key
xpack.security.transport.ssl.certificate: certs/es01/es01.crt
xpack.security.transport.ssl.certificate_authorities: certs/ca/ca.crt
## http ssl
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.key: certs/es01/es01.key
xpack.security.http.ssl.certificate: certs/es01/es01.crt
xpack.security.http.ssl.certificate_authorities: certs/ca/ca.crt
```

Após alterar a configuração nos **`3`** nós, podemos iniciar cada um.

```bash
$ sudo systemctl start elasticsearch
```
### Criando senha para os usuários de sistema<a name="resetuser"></a>

Com o cluster no ar precisamos agora de um usuário para podermos nos autenticar, sendo assim iremos resetar a senha do usuário de sistema `elastic`.

```bash
$ sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password --url "https://es01:9200" -u elastic
```

Esse comando irá se conectar no Elasticsearch do nó `es01` e resetar a senha do usuário `elastic` para um valor aleatório que será exibido na tela, caso deseje escolher uma senha, basta utilizar o parâmetro `-i` no final do comando.

Com o usuário e senha iremos validar o estado do cluster utilizando o `curl`, mas como estamos utilizando uma CA local precisaremos passar o parâmetro `-k` ou apontar para o certificado da CA.

```bash
$ sudo curl -XGET https://es01:9200/_cluster/health?pretty -u elastic:SENHA -k
```

ou

```bash
$ sudo curl -XGET https://es01:9200/_cluster/health?pretty -u elastic:SENHA --cacert /etc/elasticsearch/certs/ca/ca.crt
```

`RESPOSTA`

```
{
  "cluster_name" : "es",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 3,
  "number_of_data_nodes" : 3,
  "active_primary_shards" : 2,
  "active_shards" : 4,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
```

### Instalando o Kibana

Para o Kibana iremos utilizar apenas um servidor, também com _Rocky Linux 8_.

```bash
$ wget https://artifacts.elastic.co/downloads/kibana/kibana-8.4.3-x86_64.rpm
$ sudo rpm -ivh kibana-8.4.3-x86_64.rpm
```

### Criando certificado para o Kibana

O certificado para o Kibana pode ser criado da mesma forma que os [certificados dos nós](#nodecert), apenas alterando o arquivo `instance.yml`.

```yaml
instances:
  - name: "kibana"
    ip: ["10.0.0.104"]
    dns: ["kibana", "kibana.lab", "kibana.lab.local"]
```

```bash
$ sudo /usr/share/elasticsearch/bin/elasticsearch-certutil cert --silent --pem \
    --ca-key /etc/elasticsearch/certs/ca/ca.key \
    --ca-cert /etc/elasticsearch/certs/ca/ca.crt --ca-pass SENHA-CHAVE-CA \
    --in /tmp/instances.yml --out kibana.zip --pass PASSPHRASE-CERTIFICADOS
```

No Kibana temos duas configurações de TLS, a da comunicação com o Elasticsearch, onde o Kibana se comporta como um cliente qualquer, e a comunicação com o usuário através do browser, assim como no caso das configurações de _transport_ e _https_ dos nós de Elasticsearch, também podemos utilizar certificados diferentes se quisermos.

### Configurando a segurança no Kibana

Na versão `8` do Elasticsearch não é mais possível utilizar o usuário `elastic` na configuração do Kibana, devemos utilizar o usuário `kibana_system` ou utilizar uma [conta de serviço][service-account].

Para utilizar a conta `kibana_system` será precisar [resetar a senha](#resetuser) da mesma forma que foi feito para o usuário `elastic`.

Nesse exemplo iremos criar uma conta de serviço para a comunicação entre o Kibana e o Elasticsearch.

```bash
$ curl -X POST "https://es01:9200/_security/service/elastic/kibana/credential/token/kibanatoken?pretty" -u elastic:senha -k
```

 `RESPOSTA`

```json
{
  "created" : true,
  "token" : {
    "name" : "kibanatoken",
    "value" : "token"
  }
}
```

Iremos adicionar o token e a passphrase do certificado no keystore do Kibana, que é criado automaticamente durante a instalação.

```bash
$ sudo sh -c 'echo "passphrase" | /usr/share/kibana/bin/kibana-keystore add elasticsearch.ssl.keyPassphrase --stdin'
$ sudo sh -c 'echo "passphrase" | /usr/share/kibana/bin/kibana-keystore add server.ssl.keyPassphrase --stdin'
$ sudo sh -c 'echo "token" | /usr/share/kibana/bin/kibana-keystore add elasticsearch.serviceAccountToken --stdin'
```

Antes de editarmos o arquivo `kibana.yml`, precisamos copiar o certificado da CA e os certificados do Kibana para o servidor do Kibana, para isso é preciso criar o caminho `/etc/kibana/certs`.

`kibana.yml`

```
server.port: 5601
server.host: "10.0.0.104"
server.publicBaseUrl: "https://kibana:5601"
server.maxPayload: 5048576
server.name: "kibana"

# elasticsearch
elasticsearch.hosts: ["https://es01:9200", "https://es02:9200", "https://es03:9200"]

elasticsearch.ssl.certificate: /etc/kibana/certs/kibana/kibana.crt
elasticsearch.ssl.key: /etc/kibana/certs/kibana/kibana.key
elasticsearch.ssl.certificateAuthorities: [ "/etc/kibana/certs/ca/ca.crt" ]

# server https
server.ssl.enabled: true
server.ssl.certificate: /etc/kibana/certs/kibana/kibana.crt
server.ssl.key: /etc/kibana/certs/kibana/kibana.key
server.ssl.certificateAuthorities: [ "/etc/kibana/certs/ca/ca.crt" ]
```

Para iniciar o Kibana usamos o `systemctl`.

```bash
$ sudo systemctl start kibana
```

Após iniciar o Kibana podemos acessar o servidor via navegador para validar que a conexão está usando TLS, será mostrado um alerta de que a conexão não é privada já que o navegador não conhece a CA local que utilizamos para gerar o certificado do Kibana, se necessário isso pode ser solucionado importando a CA local para o navegador utilizado.

![kibana-cert](/img/posts/0008/0008-04.png)

### Mais informações

Mais informações sobre como configurar a Segurança no Elasticsearch e outras formas de autenticação disponíveis dependendo da licença em uso podem ser encontradas na [documentação oficial][security-doc].

[security-free]: https://www.elastic.co/blog/security-for-elasticsearch-is-now-free
[service-account]: https://www.elastic.co/guide/en/elasticsearch/reference/current/service-accounts.html#service-accounts-explanation
[security-guide]: https://www.elastic.co/guide/en/elasticsearch/reference/current/manually-configure-security.html
[security-doc]: https://www.elastic.co/guide/en/elasticsearch/reference/current/secure-cluster.html