+++
description = "Quebrando o master password de um vault do 1Password usando o Hashcat"
categories = [
	"segurança", "1password", "hashcat", "crypto"
]
title = "Quebrando um vault 1Password usando o hashcat"
date = "2016-10-27T11:39:38-02:00"

+++

Imaginem a seguinte situação: começar a usar um gerenciador de senhas, cadastrar várias informações nele e depois de um tempo, esquecer a senha principal, a `master password`, a única senha que deveríamos lembrar. 

Pois é, aconteceu lá em casa <span>&#x1F614;</span>

Usando o [1Password](https://1password.com/) pelo celular, o acesso ao `vault` era feito pelo `TouchID` até reiniciar o dispositivo, quando precisamos colocar a `master password` novamente.

Enfim, precisava uma forma automatizada de testar várias possibilidades de senha para recobrar o acesso ao `vault`. 

A primeira tentativa foi usando o cliente Java que fiz para acessar o `1Password` via linha de comando, [1pass](https://github.com/luishgo/1pass). Gerei uma lista de possíveis senhas e rodei o cliente várias vezes sem sucesso. Além da lentidão, precisava gerar todas as possibilidades, sem o uso de máscaras.

Deixei isso de lado até me deparar com o [hashcat](https://hashcat.net/hashcat/). `Open-source`, rápido (usa a GPU para processamento), suporta máscaras e os formatos de `vaults` do `1Password`. Era só preparar a entrada e as máscaras para rodar na versão `3.10` e torcer.

## Hash de entrada

Não basta apontar o `hashcat` para a pasta do `vault`, é preciso extrair os dados do arquivo `encryptionKeys.js` para um formato de `hash` aceito pelo programa. O formato aceito é `iterações:salt:data` com `salt`  e `data` em hexadecimal. Usei um [script perl](https://github.com/philsmd/1password_agilekeychain_to_hashcat) feito por um dos desenvolvedores do `hashcat`, `philsmd`, para realizar a extração e salvar em um arquivo `.hash`.

## Máscaras

O próximo passo foi criar um arquivo `.hcmask` contendo várias máscaras para ampliar as chances de encontrar a senha certa em um tempo decente. O `hashcat` tem conjuntos de caracteres padrão para explicitar as máscaras:

* `?l = abcdefghijklmnopqrstuvwxyz`
* `?u = ABCDEFGHIJKLMNOPQRSTUVWXYZ`
* `?d = 0123456789`
* `?s = «space»!"#$%&'()*+,-./:;<=>?@[\]^_```{|}~`
* `?a = ?l?u?d?s`
* `?b = 0x00 - 0xff`

Juntei os trechos conhecidos da senha, ou o mais próximo disso que tinha, com caracteres curinga usando os conjuntos padrão e fiquei com um arquivo `.hcmask` com o seguinte formato:

```
parteconhecida?a?a?a
senha?a?a
10?a?a2016
```

## Execução

`hashcat64.exe -m 6600 -a 3 tmp.hash tmp.hcmask`, onde

* `-m 6600`: modo 1Password, agilekeychain
* `-a 3`: modo de ataque com máscaras
* `tmp.hash`: saída do script perl com `hash` de entrada
* `tmp.hcmaks`: máscaras a serem testadas

Resultado final

```
Session.Name...: hashcat
Status.........: Cracked
Input.Mode.....: Mask (?a?a?a) [3]
Hash.Target....: File (tmp.hash)
Hash.Type......: 1Password, agilekeychain
Time.Started...: Thu Oct 27 16:22:16 2016 (1 min, 9 secs)
Speed.Dev.#1...:     2611 H/s (2.42ms)
Recovered......: 2/2 (100.00%) Digests, 2/2 (100.00%) Salts
Progress.......: 947625/1714750 (55.26%)
Rejected.......: 0/947625 (0.00%)
Restore.Point..: 0/9025 (0.00%)

Started: Thu Oct 27 16:22:16 2016
Stopped: Thu Oct 27 16:23:30 2016
```