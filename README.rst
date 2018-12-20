WebAPI com ASP.NET Core
=======================


Para criar o template do projeto execute:

.. code:: bash

   $ dotnet new webapi -o 'nome-do-projeto'
   $ cd 'nome-do-projeto'

O Kestrel não vai executar de primeira porque não temos um certificado para
servir páginas via SSL.  Isso significa que não podemos servir via HTTPS.

Podemos gerar um certificado para desenvolvermos no ``localhost`` usando o OpenSSL_.
Para isso devemos, antes de tudo, criar um arquivo de configuração do OpenSSL
especificando as informações do certificado em um ponto ``.conf``.

.. _OpenSSL: https://www.openssl.org/

O ``localhost.conf`` tem este formato:

.. code:: dosini

   [ req ]
   prompt              = no
   default_bits        = 2048
   default_keyfile     = localhost.pem
   distinguished_name  = subject
   req_extensions      = req_ext
   x509_extensions     = x509_ext
   string_mask         = utf8only

   # The Subject DN can be formed using X501 or RFC 4514 (see RFC 4519 for a description).
   # Its sort of a mashup. For example, RFC 4514 does not provide emailAddress.
   [ subject ]
   countryName         = BR
   stateOrProvinceName = Distrito Federal
   localityName        = Distrito Federal
   organizationName    = Sua Empresa Inc.


   # Use a friendly name here because its presented to the user. The server's DNS
   # names are placed in Subject Alternate Names. Plus, DNS names here is deprecated
   # by both IETF and CA/Browser Forums. If you place a DNS name here, then you 
   # must include the DNS name in the SAN too (otherwise, Chrome and others that
   # strictly follow the CA/Browser Baseline Requirements will fail).
   commonName          = Localhost dev cert
   emailAddress        = seu@email.com

   # Section x509_ext is used when generating a self-signed certificate. I.e., openssl req -x509 ...
   [ x509_ext ]

   subjectKeyIdentifier    = hash
   authorityKeyIdentifier  = keyid,issuer

   # You only need digitalSignature below. *If* you don't allow
   # RSA Key transport (i.e., you use ephemeral cipher suites), then
   # omit keyEncipherment because that's key transport.
   basicConstraints    = CA:FALSE
   keyUsage            = digitalSignature, keyEncipherment
   subjectAltName      = @alternate_names
   nsComment           = "OpenSSL Generated Certificate"

   # RFC 5280, Section 4.2.1.12 makes EKU optional
   # CA/Browser Baseline Requirements, Appendix (B)(3)(G) makes me confused
   # In either case, you probably only need serverAuth.
   # extendedKeyUsage  = serverAuth, clientAuth

   # Section req_ext is used when generating a certificate signing request. I.e., openssl req ...
   [ req_ext ]

   subjectKeyIdentifier    = hash

   basicConstraints        = CA:FALSE
   keyUsage                = digitalSignature, keyEncipherment
   subjectAltName          = @alternate_names
   nsComment               = "OpenSSL Generated Certificate"

   # RFC 5280, Section 4.2.1.12 makes EKU optional
   # CA/Browser Baseline Requirements, Appendix (B)(3)(G) makes me confused
   # In either case, you probably only need serverAuth.
   # extendedKeyUsage  = serverAuth, clientAuth

   [ alternate_names ]
   DNS.1       = localhost

   # Add these if you need them. But usually you don't want them or
   # need them in production. You may need them for development.
   # DNS.5       = localhost
   # DNS.6       = localhost.localdomain
   # DNS.7       = 127.0.0.1

   # IPv6 localhost
   # DNS.8     = ::1

Agora você pode gerar as credenciais (``localhost.{key,crt,pfx}``):

.. code:: bash

   $ openssl req -config localhost.conf -new -x509 -sha256 -newkey rsa:2048 -nodes -keyout localhost.key -days 3650 -out localhost.crt
   $ openssl pkcs12 -export -out localhost.pfx -inkey localhost.key -in localhost.crt


O Kestrel deve usar o ``localhost.pfx`` para servir a aplicação via HTTPS.
Também devemos importar as chaves e certificados para o PKCS #12 e o banco de dados do NSS [#]_:

.. code:: bash

   $ pk12util -d sql:$HOME/.pki/nssdb -i localhost.pfx
   $ certutil -d sql:$HOME/.pki/nssdb -A -t "P,," -n 'dev cert' -i localhost.crt

(A parte anterior eu tirei do blog `.NET Escapades`_ do Andrew Lock, com alterações mínimas.)

.. _.NET Escapades: https://andrewlock.net/creating-and-trusting-a-self-signed-certificate-on-linux-for-use-in-kestrel-and-asp-net-core/

Feito isso, mude o programa ``Program.cs`` para executar o Kestrel com os parâmetros
de porta modificados:

.. code:: csharp

   using System;
   using System.Net; // Necessário para usarmos o `IPAddress`
   using System.Collections.Generic;
   using System.IO;
   using System.Linq;
   using System.Threading.Tasks;
   using Microsoft.AspNetCore;
   using Microsoft.AspNetCore.Hosting;
   using Microsoft.Extensions.Configuration;
   using Microsoft.Extensions.Logging;

   namespace TodoApi
   {
       public class Program
       {
           public static void Main(string[] args)
           {
               CreateWebHostBuilder(args).Build().Run();
           }

           public static IWebHostBuilder CreateWebHostBuilder(string[] args) =>
               WebHost.CreateDefaultBuilder(args)
                   ////////// Configurando o Kestrel //////////
                   .UseKestrel(options =>
                   {
                       options.Listen(IPAddress.Loopback, 5001);
                       options.Listen(IPAddress.Loopback, 5002, listenOptions =>
                       {
                           listenOptions.UseHttps("../localhost-cert/localhost.pfx", "testpassword");
                       });
                   })
                   ////////// Fim da configuração //////////
                   .UseStartup<Startup>();
       }
   }

--------------------------------------------------------------------------------

Agora vá ser feliz!!!

.. [#] Se você não conseguir executar estes comandos então instale os pacotes do ``libnss3``.
   No Arch Linux é só executar ``pacman -S nss``.

