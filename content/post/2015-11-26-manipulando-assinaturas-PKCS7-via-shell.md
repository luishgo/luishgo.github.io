---
date: 2015-11-26T00:00:00Z
title: Manipulando assinaturas PKCS#7 via Shell
url: /2015/11/26/manipulando-assinaturas-pkcs7-via-shell/
---

Esses tempos, estava com um conjunto de mais de 1000 assinaturas inválidas. Eram de diferentes usuários, seria muito complicado refazê-las. Por um problema na geração do pacote PKCS#7, a data salva no atributo `signingTime` estava adiantada alguns segundos, invalidando a assinatura em questão.

Para corrigi-las, precisei descobrir como manipular os atributos de uma assinatura PKCS#7. No caso, precisava encontrar o valor de `signingTime` correto para tornar a assinatura válida. Sabendo que o valor estava adiantado e o atributo tem precisão de segundos, a proposta de solução foi de ir atrasando 1 segundo no `signingTime` até encontrar o valor onde o pacote PKCS#7 ficaria válido.

A ideia da solução foi usar apenas comando do shell para praticar. No fim das contas, foi preciso um pequeno script em Perl para decrementar um segundo da data extraída do `signingTime`.

A implementação da solução foi dividida nos passos abaixo:

Verificar a assinatura
---

O primeiro passo foi verificar uma assinatura PKCS#7 detached usando o openssl:

`openssl smime -verify -inform <DER ou PEM> -in <arquivo da assinatura> -content <arquivo assinado> -noverify`

Apesar de parecerem contraditórios, `-verify` manda a assinatura ser verificada e o `-noverify` faz com que os certificados usados na assinatura não sejam verificados. Assim, apenas os atributos assinados são utilizados na verificação.

Encontrar o *offset* do atributo `signingTime`
---

Dentro de um pacote PKCS#7, pode-se encontrar mais de um `signingTime` após a execução do `openssl asn1parse`:

`openssl asn1parse -in <arquivo da assinatura> | grep signingTime --after-context=2`

Exemplo de resultado:

<pre>2233:d=7  hl=2 l=   9 prim: OBJECT            :signingTime
2244:d=7  hl=2 l=  15 cons: SET
2246:d=8  hl=2 l=  13 prim: UTCTIME           :150826172903Z
--
5162:d=15 hl=2 l=   9 prim: OBJECT            :signingTime
5173:d=15 hl=2 l=  15 cons: SET
5175:d=16 hl=2 l=  13 prim: UTCTIME           :150826173747Z
</pre>

O `--after-context=2` ou `-A 2` apresenta duas linhas após a linha encontrada com o padrão. 

O primeiro `signingTime` é o relevante, neste caso. Para apresentar apenas a linha correta, adicionei:

`head -3 | tail -1`

O `head -3` retorna apenas as três primeiras linhas e o `tail -1` retorna apenas a última linha.

Agora, basta extrair o primeiro número da linha e somar o tamanho do cabeçalho, informado na própria linha, `hl=2`:

`awk -F ':' '{print $1+2}'`

O `-F` determina o separador de campos. O valor a ser somado, `+2`, é referente ao tamanho do cabeçalho do campo e até poderia ser extraído da linha, mas como é um valor conhecido e fixo no pacote, não valia a pena o esforço.

Tudo junto ficou:

`openssl asn1parse -in <arquivo da assinatura> | grep signingTime -A 2 | head -3 | tail -1 | awk -F ':' '{print $1+2}'`

Extrair o valor do `signingTime`
---

De posse do *offset*, o comando `dd` permitiu extrair o valor do `signingTime`:

`dd if=<arquivo da assinatura> bs=1 count=13 skip=<offset encontrado> status=none`

A opção `bs` define o tamanho do bloco e a opção `count` define a quantidade desses blocos para copiar. No caso, os 13 bytes do arquivo de assinatura a partir da posição determinada pelo `skip`. O `status=none` suprime todas as informações de transferência e etc. que o `dd` normalmente emite, sobrando apenas o valor do `signingTime`. 

Alterar o valor do `signingTime`
---

Nesse ponto, os meus conhecimentos em shell fraquejaram e precisei apelar para um pequeno script em perl para decrementar X segundos do `signingTime`:

<pre>
#!/usr/bin/env perl
use strict;
use warnings;
use Time::Piece;

print subtract($ARGV[0], $ARGV[1]);

sub subtract {
  my ($date, $seconds)=@_;
  my $t = Time::Piece->strptime($date, "%y%m%d%H%M%SZ");
  $t-=$seconds;
  return $t->strftime("%y%m%d%H%M%SZ");
}
</pre>

Para implementar isso em shell, teria que descobrir como interpretar a data no formato do `signingTime` antes de poder alterá-la. Complicado :P.

O script apenas retorna o novo valor de data, sendo necessário colocá-lo no arquivo de assinatura na posição correta. Mais uma vez, o `dd` faz o trabalho:

`dd of=<arquivo da assinatura> bs=1 seek=<offset encontrado> count=13 conv=notrunc status=none`

O `seek` entra no lugar do `skip` quando tratamos de uma operação de escrita.

Juntando tudo
---

Com os passos anteriores testados, bastou juntar tudo para descobrir o valor correto de `signingTime` para cada assinatura problemática:

<pre>
FILE=$1
SIGNATURE=$2

OFFSET=`openssl asn1parse -inform DER -in $SIGNATURE -i | grep signingTime -A 2 | head -3 | tail -1 | awk -F ':' '{print $1+2}'`
UTCTIME=`dd if=$SIGNATURE bs=1 count=13 skip=$OFFSET status=none`
FOUND=false
for i in $(seq 0 1000)
do       
  perl time.pl $UTCTIME $i | dd of=$SIGNATURE bs=1 seek=$OFFSET count=13 conv=notrunc status=none 
  if openssl smime -verify -inform DER -in $SIGNATURE -content $FILE -noverify 2>&1 > /dev/null | grep "Verification successful"
  then
  	echo $SIGNATURE $i
  	FOUND=true
    break
  fi
done
if [ "$FOUND" = false ]
then
	echo $SIGNATURE "Não encontrada"
fi
</pre>