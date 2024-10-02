+++
title = 'Escanenado arquivos JavaScript com a ProjectDiscovery'
date = 2024-03-28T16:25:49-03:00
draft = false
description = "Um guia simples e prático de como podemos automatizar o escaneamento de arquivos JavaScript com ferramentas da ProjectDiscovery"
tags = ['pentest', 'javascript', 'automation', 'bug bounty']
+++

## Introdução
Para vocês que talvez não saibam, **ProjectDiscovery** é a equipe responsável pelo desenvolvimento de diversas ferramentas como Nuclei, Httpx, Katana, DnsX, entre outras. Gosto bastante de todo o projeto, e a intenção de facilitar e automatizar alguns processos de pentest é realmente útil. Utilizo o Nuclei, Httpx e outros projetos deles há bastante tempo, e tudo sempre funcionou bem.

Basicamente, vou demonstrar como você pode combinar algumas ferramentas da ProjectDiscovery para automatizar a captura e o escaneamento de JavaScripts em busca de segredos (chaves de API, senhas e afins), combinando o Subfinder, Httpx, Katana e Nuclei (nuclei-templates).

{{< figure src="https://compote.slate.com/images/d821ec33-6e11-4cf1-a5e1-64e48f0173b7.jpeg" >}}

___

## Preparando o ambiente

Para começar, você precisa instalar todas as ferramentas que iremos utilizar. Considerando que você já tem o Go instalado em sua máquina, com uma versão superior a 1.21, aqui está a lista de comandos para instalar as ferramentas:

- Nuclei
```
go install -v github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest
```

- Subfinder
```
go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest
```

- Httpx
```
go install -v github.com/projectdiscovery/httpx/cmd/httpx@latest
```

- Katana
```
go install -v github.com/projectdiscovery/katana/cmd/katana@latest
```

Certifique-se de que a variável de ambiente `GOPATH` está definida corretamente para que você possa executar as ferramentas diretamente pelo seu terminal.

> Para instalar o `nuclei-templates`, é necessário executar a ferramenta nuclei pelo menos uma vez.

___

## O início

Após instalar todas as ferramentas, podemos começar a utilizá-las para fazer um pouco de magia. Assim, precisamos enumerar os subdomínios do nosso alvo para depois capturar todos os endpoints. Estarei utilizando o Subfinder para isso. Recomendo que vocês configurem algumas [chaves de API que o Subfinder suporta](https://highon.coffee/blog/subfinder-cheat-sheet/); isso vai ajudar a encontrar mais subdomínios.

O comando que iremos utilizar é **`subfinder -all -d target.domain -o domains.txt`**. Podemos esperar um resultado parecido com isso:
![subfinder](https://i.imgur.com/JaUc6vx.png)

Após enumerar todos os subdomínios e salvar a saída do comando em um arquivo de texto, precisamos verificar quais estão ativos ou não. Você pode utilizar esse comando: **`httpx -l domains.txt -o domains.httpx`**

*Dica: Você pode especificar portas diferentes como 8080, 8081, 81 e outras para encontrar outros serviços que rodam sobre o mesmo ativo.*

{{< figure src="https://i.imgur.com/OGKJUFH.png" >}}

Esse é um dos principais processos que utilizaremos para capturar os arquivos JavaScript. Vamos usar o Katana para capturar todos os endpoints dos subdomínios encontrados, utilizando o atributo `-jc`. Eu usarei esse comando, mas você pode aprimorá-lo com o grep e outros binários: **`cat domains.httpx | katana -jc -o endpoints.txt`**.

{{< figure src="https://i.imgur.com/pOE0fc3.png" >}}

Com todos os endpoints capturados, filtraremos aqueles que terminam com **.js**. Para isso, utilizaremos **`grep '.js' endpoints.txt > js_files.txt`**. No entanto, isso também armazenará os endpoints que terminam com .json. Para removê-los, podemos utilizar o grep novamente: **`grep -v '.json' js_files.txt > js_files_cleaned.txt`**.

{{< figure src="https://i.imgur.com/uaQUI3v.png" >}}
{{< figure src="https://i.imgur.com/acyk9Ou.png" >}}

Depois de filtrar os endpoints, precisamos salvar os arquivos JavaScript em algum lugar. Vamos combinar os binários **cURL** e **xargs** para salvá-los em um diretório. Podemos utilizar o *awk* para extrair o nome do domínio e o nome do arquivo, facilitando a identificação posteriormente. Antes de executar o comando abaixo, certifique-se de que você está em um diretório apropriado para armazenar todos os arquivos JavaScript.
```
while read url; do
  filename=$(echo $url | awk -F/ '{gsub(/[\/:]/, "-", $3); print $3 "-" $NF}')
  curl "$url" > "${filename}.js"
done < ../js_files_cleaned.txt
```

{{< figure src="https://i.imgur.com/CErGIC0.png" >}}

___

## O fim

O resultado final do comando será algo parecido com o seguinte:

{{< figure src="https://i.imgur.com/VIvgXOx.png" >}}

Agora, para encontrarmos possíveis vazamentos de segredos (secrets) no código, utilizaremos o Nuclei juntamente com os nuclei-templates. Este é o comando que venho utilizando, e sempre funcionou para mim:
```
find <js_files_path> -type f -name "*.js" | while read jsfile; do nuclei -t ~/nuclei-templates/file/ -target "$jsfile" -severity low,medium,high,critical -et ~/nuclei-templates/file/xss/dom-xss.yaml; done
```

{{< figure src="https://i.imgur.com/TjDh4oE.png" >}}
{{< figure src="https://i.imgur.com/a5hZ5ql.png" >}}

___
## Avisos e Declarações
- Você provavelmente será bloqueado pelo WAF (Web Application Firewall), então tenha cuidado.
- Naturalmente, haverá muitos falsos positivos, portanto, seja paciente e verifique cada resultado para determinar o que é verdadeiro ou não.
- Existem, provavelmente, outras maneiras melhores de realizar essa tarefa, mas acredito que esta também seja eficaz. Use essa se quiser.
- Não estou compartilhando isso para provar qualquer ponto específico, apenas acredito que possa ser útil para pessoas interessadas em experimentar abordagens diferentes.

___

## Referências
- [Subfinder Cheatsheet](https://highon.coffee/blog/subfinder-cheat-sheet/)
- [ProjectDiscovery](https://github.com/projectdiscovery)
- [Davis Sopas -  Using katana to get API endpoints](https://www.youtube.com/watch?v=9d1D-DB6BkE)

### Tags