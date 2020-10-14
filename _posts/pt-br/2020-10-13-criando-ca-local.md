---
layout: post
title: "criando uma autoridade certificadora local"
date: 2020-10-13 21:30:00 -0300
lang: pt-br
ref: "p0003"
categories: [ posts, pt-br ]
tags: [ infra ]
cover: /img/posts/covers/0003.png
---
### Introdução

Embora hoje em dia seja extremamente simples criar na nuvem um ambiente temporário para estudos ou testes, eu ainda sou um pouco _old school_ e tenho um pequeno computador que utilizo como laboratório, nada muito complexo, um antigo _Dell OptiPlex Micro_ com **Core i5**, **16 GB** de RAM e **500 GB** de HD.

Nesse pequeno servidor eu costumo criar máquinas virtuais utilizando o **KVM**/**Virt-Manager**, ou containers utilizando o **Docker**, onde rodo serviços e aplicações que quero aprender como funcionam, fazer provas de conceito e realizar alguns testes com aplicações que já utilizo tanto no trabalho quanto em projetos pessoais.

Como estou em um ambiente controlado, acabo negligenciando um pouco as questões relacionadas a segurança e não utilizo certificados _SSL/TLS_ nos serviços e na comunicação entre eles, o que é algo que não vai acontecer no mundo real e frequentemente exige um trabalho extra pra validar as configurações e funcionamento com certificados na hora de implementar em produção.

Uma forma simples pra se acostumar a trabalhar sempre com certificados é criar uma Autoridade Certificadora local para criar e assinar certificados para as aplicações e serviços.

### Criando uma Autoridade Certificadora local

O processo pra criar uma autoridade certificadora é bem simples e consiste basicamente de dois passos, o único requisito é ter o `openssl` instalado na máquina.

1. Criar uma chave privada pra a Autoridade Certificadora. 
2. Criar um certificado _root_ utilizando essa chave privada.

Criamos a chave privada com o seguinte comando.

```
# openssl genrsa -des3 -out localCA.key 2048
```

A execução do comando solicita que seja criada uma senha para a chave privada, essa mesma senha será solicitada na criação do certificado _root_.

Para criar o certificado _root_ utilizamos o seguinte comando.

```
# openssl req -x509 -new -key localCA.key -sha256 -days 7300 -out localCA.pem
```

No comando acima o parâmetro `-x509` indica que queremos gerar um certificado _root_ auto-assinado e em conjunto com o parâmetro `-days` indica por quantos dias esse certificado será válido, além disso a execução do comando faz uma série de perguntas, como por exemplo o código de duas letras do país, o estado, a cidade etc.

Após esses dois comandos temos dois arquivos:

- `localCA.key` - chave privada da autoridade certificadora local.
- `localCA.pem` - certificado _root_ da autoridade certificadora local.

Esses dois arquivos são necessários para assinar os certificados de serviços e aplicações.

### Criando certificados para serviços e aplicações

Agora que já temos uma autoridade certificadora local, podemos criar certificados para serviços e aplicações, no caso vamos criar um certificado para um servidor web que vai responder no hostname local `dev.lab`.

O procedimento é bem semelhante a criação da chave privada e do certificado _root_.

Primeiro criamos uma chave privada para a aplicação.

```
# openssl genrsa -out dev.lab.key 2048
```

Na sequência utilizamos essa chave para criar um certificado do tipo _csr_, _certificate signing request_, ou seja uma solicitação de assinatura para o certificado.

```
# openssl req -new -key dev.lab.key -out dev.lab.csr
```

Novamente serão feitas algumas perguntas como o código de duas letras do país, estado, cidade etc.

Antes de assinarmos esse certificado com a autoridade certificadora local, precisamos criar um arquivo auxiliar com as informações de **DNS** do servidor web.

Vamos ciar o arquivo `dev.lab.ext` com o seguinte conteúdo

```
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names
 
[alt_names]
DNS.1 = dev.lab
```

Agora já podemos solicitar que a autoridade certificadora assine o certificado da aplicação com o seguinte comando.

```
# openssl x509 -req -in dev.lab.csr -CA localCA.pem -CAkey localCA.key -CAcreateserial -out dev.lab.crt -days 7300 -sha256 -extfile dev.lab.ext
```

Será novamente solicitada a senha da chave privada da autoridade certificadora local.

Com os arquivos `dev.lab.crt` e `dev.lab.key` podemos configurar o servidor web para utilizar o certificado, no caso do **NGINX** usamos a configuração padrão.

```
server {
    listen       443 ssl http2 default_server;
    listen       [::]:443 ssl http2 default_server;
    server_name  dev dev.lab;
    root         /usr/share/nginx/html;
    ssl_certificate "/etc/nginx/certs/dev.lab.crt";
    ssl_certificate_key "/etc/nginx/certs/dev.lab.key";
    ssl_session_cache shared:SSL:1m;
    ssl_session_timeout  10m;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    include /etc/nginx/default.d/*.conf;

    location / {
    }

    error_page 404 /404.html;
    location = /404.html {
    }

    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
    }
}
```

### Validando o certificado

Acessando o hostname `dev.lab` no navegador temos a mensagem de que a conexão não é privada.

![erro no certificado](/img/posts/0003-01.png)

Recebemos essa mensagem porque, embora o servidor esteja utilizando um certificado, o navegador não reconhece a autoridade certificadora que assinou esse certificado.

Podemos resolver esse problema importando o certificado _root_, `localCA.pem`, que criamos anteriormente diretamente no navegador.

Após a importação do certificado o navegador passa a aceitar certificados assinados pela autoridade certificadora local.

![certificado ok](/img/posts/0003-02.gif)