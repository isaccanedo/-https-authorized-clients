HTTPS Authorized Certs
======================

Normalmente, os servidores HTTPS fazem um handshake básico de TLS e aceitam qualquer conexão de cliente como
desde que um pacote de criptografia compatível possa ser encontrado. No entanto, o servidor pode ser configurado
para enviar ao cliente um CertificateRequest durante o handshake TLS que requer o
cliente para apresentar um certificado como forma de identidade.

Aqui estão algumas informações sobre
[Handshakes TLS autenticados pelo cliente](http://en.wikipedia.org/wiki/Transport_Layer_Security#Client-authenticated_TLS_handshake)
at Wikipedia.

Os certificados do servidor HTTPS geralmente têm seu "Nome comum" definido como seu nome de domínio totalmente qualificado e são assinados por uma autoridade de certificação bem conhecida, como a Verisign.
No entanto, o "Nome comum" geralmente usado em certificados de cliente pode ser definido como qualquer coisa que identifique o cliente, como "Acme, Co." ou "client-12345". Isso será apresentado ao servidor e pode ser usado em adição ou em vez de estratégias de nome de usuário / senha para
identificar o cliente.

Usando node.js, pode-se instruir o servidor a solicitar um certificado de cliente e rejeitar clientes não autorizados adicionando

    {
      requestCert: true,
      rejectUnauthorized: true
    }

às opções passadas para https.createServer (). Por sua vez, um cliente será rejeitado a menos que passe um certificado válido em suas opções https.request().

    {
      key: fs.readFileSync('keys/client-key.pem'),
      cert: fs.readFileSync('keys/client-crt.pem')
    }

O exercício a seguir criará uma autoridade de certificação autoassinada, um certificado de servidor e dois certificados de cliente, todos "autoassinados" pela autoridade de certificação. Em seguida, executaremos um servidor HTTPS que aceitará apenas conexões feitas por clientes que apresentem um certificado válido.
Terminaremos revogando um dos certificados do cliente e vendo que o servidor rejeita solicitações deste cliente.

Setup
=====

Vamos criar nossa própria autoridade de certificação para que possamos assinar nossos próprios certificados de cliente. Também assinaremos nosso certificado de servidor para que não tenhamos que pagar por um para nosso servidor.

Crie uma autoridade de certificação
------------------------------

Faremos isso apenas uma vez e usaremos a configuração armazenada em keys/ca.cnf. Um certificado de 27 anos (9999 dias) de 4096 bits deve fazer o truque muito bem. (queremos que nosso CA seja válido por um longo tempo e seja super seguro - mas isso é realmente um exagero)

    openssl req -new -x509 -days 9999 -config keys/ca.cnf -keyout keys/ca-key.pem -out keys/ca-crt.pem

Agora temos uma autoridade de certificação com a chave privada keys/ca-key.pem e chave pública 
keys/ca-crt.pem.

Criar chaves privadas
-------------------

Vamos construir algumas chaves privadas para nossos certificados de servidor e cliente.

    openssl genrsa -out keys/server-key.pem 4096
    openssl genrsa -out keys/client1-key.pem 4096
    openssl genrsa -out keys/client2-key.pem 4096

Novamente, 4096 é um pouco exagerado aqui, mas não estamos muito preocupados com os problemas de uso da CPU.

Certificados de assinatura
-----------------

Agora, vamos assinar esses certificados usando a autoridade de certificação que criamos anteriormente. Isso geralmente é chamado de "auto-assinatura" de nossos certificados. Começaremos assinando o certificado do servidor.

    openssl req -new -config keys/server.cnf -key keys/server-key.pem -out keys/server-csr.pem
    openssl x509 -req -extfile keys/server.cnf -days 999 -passin "pass:password" -in keys/server-csr.pem -CA keys/ca-crt.pem -CAkey keys/ca-key.pem -CAcreateserial -out keys/server-crt.pem

A primeira linha cria um "CSR" ou pedido de assinatura de certificado que é escrito em keys / server-csr.pem
Em seguida, usamos a configuração armazenada em keys / server.cnf e nossa autoridade de certificação para assinar o CSR
resultando em keys / server-crt.pem, o novo certificado público de nosso servidor.

Vamos fazer o mesmo para os dois certificados de cliente, usando arquivos de configuração diferentes. (os arquivos de configuração são idênticos, exceto para a configuração do nome comum para que possamos distingui-los mais tarde)

    openssl req -new -config keys/client1.cnf -key keys/client1-key.pem -out keys/client1-csr.pem
    openssl x509 -req -extfile keys/client1.cnf -days 999 -passin "pass:password" -in keys/client1-csr.pem -CA keys/ca-crt.pem -CAkey keys/ca-key.pem -CAcreateserial -out keys/client1-crt.pem

    openssl req -new -config keys/client2.cnf -key keys/client2-key.pem -out keys/client2-csr.pem
    openssl x509 -req -extfile keys/client2.cnf -days 999 -passin "pass:password" -in keys/client2-csr.pem -CA keys/ca-crt.pem -CAkey keys/ca-key.pem -CAcreateserial -out keys/client2-crt.pem

OK, devemos receber os certificados de que precisamos.

Verificar
------

Vamos apenas testá-los para ter certeza de que cada um desses certificados foi assinado de forma válida por nossa autoridade de certificação.

    openssl verify -CAfile keys/ca-crt.pem keys/server-crt.pem
    openssl verify -CAfile keys/ca-crt.pem keys/client1-crt.pem
    openssl verify -CAfile keys/ca-crt.pem keys/client2-crt.pem

Se obtivermos um "OK" ao executar cada um desses comandos, está tudo pronto.

Execute o exemplo
===============

Devemos estar prontos para ir agora. Vamos ligar o servidor:

    node server

Agora temos um servidor escutando 0.0.0.0:4433 que só funcionará se o cliente apresentar um certificado válido assinado pela autoridade de certificação. Vamos testar isso em outra janela:

    node client 1

Isso invocará um cliente usando o certificado client1-crt.pem, que deve se conectar ao servidor e obter um "hello world" de volta no corpo. Vamos tentar também com o outro certificado de cliente:

    node client 2

Você deve ser capaz de ver na saída do servidor que ele pode distinguir entre os dois clientes pelos certificados que eles apresentam. (client1 ou client2 que são os nomes comuns definidos nos arquivos .cnf)

Revogação de certificado
======================

Tudo está bem no mundo até que queremos encerrar um cliente específico sem encerrar todos os outros e regenerar os certificados. Vamos criar uma lista de revogação de certificado (CRL) e revogar o certificado client2. Na primeira vez que faremos isso, precisamos criar um banco de dados vazio:

    touch ca-database.txt

Agora vamos revogar o certificado do cliente2 e atualizar a CRL:

    openssl ca -revoke keys/client2-crt.pem -keyfile keys/ca-key.pem -config keys/ca.cnf -cert keys/ca-crt.pem -passin 'pass:password'
    openssl ca -keyfile keys/ca-key.pem -cert keys/ca-crt.pem -config keys/ca.cnf -gencrl -out keys/ca-crl.pem -passin 'pass:password'

Vamos parar o servidor e comentar na linha 8 que lê na CRL:

    crl: fs.readFileSync('keys/ca-crl.pem')

e reinicie o servidor novamente:

    node server

Agora é a hora da verdade. Vamos testar para ver se o cliente 2 funciona ou não:

    node client 2

Se tudo correr bem, não funcionará mais. Apenas como verificação de integridade, vamos garantir que o cliente 1 ainda funcione:

    node client 1

Da mesma forma, se tudo estiver bem, o cliente 1 ainda funciona enquanto o cliente 2 é rejeitado.

Conclusion
==========

Vimos como podemos criar certificados de cliente e servidor autoassinados e garantir que os clientes que interagem com nosso servidor usem apenas certificados válidos assinados por nós. Além disso, podemos revogar qualquer um dos certificados do cliente sem ter que revogar tudo e reconstruir do zero.
Como podemos ver o nome comum dos certificados do cliente sendo apresentados e sabemos que eles devem ser válidos para que possamos vê-los, podemos usar isso como uma estratégia para identificar clientes usando nosso servidor.
