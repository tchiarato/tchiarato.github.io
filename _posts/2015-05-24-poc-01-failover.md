---
layout: post
title: "#1 Prove of Concept: Failover com Ruby e MongoDB"
description: "Veja na prática como o MongoDB se recupera de falhas (failover)"
categories:
- Ruby
- MongoDB
tags: [ruby, MongoDB]
permalink: failover-com-ruby-e-mongodb
comments: true
---

Quando vi pela primeira vez como funciona o sistema de recuperação de falhas do
MongoDB (**Replica Set**) foi como se um mundo novo tivesse se aberto.

Veja com esse POC, como funciona na prática o processo de substituição  de nós e
recuperação de falhas do MongoDB.

## O que é Failover e Replica Set?

Failover é a forma com que um sistema pode se recuperar de uma falha sendo
substituido por um systema secundário.

Um replset ou ReplicaSet pode ser descrito como um processo de sincronização de
dados em múltiplos servidores. Este processo de replicação aumenta a capacidade
que o sistema tem de se recuperar de falhas e perda de dados.

Você pode saber mais, acessando o post que eu escrevi para o Awesome blog [Ship
It](http://shipit.resultadosdigitais.com.br/blog/alta-disponibilidade-e-tolerancia-a-falhas-com-mongodb/) da [Resultados Digitais](http://www.resultadosdigitais.com.br).

___

Para realizar este POC, eu criei um script Ruby que realiza um loop infinito lendo
os dados de um documento do MongoDB.

Todo o código que eu usei pode ser encontrado no meu
[Github](http://github.com/tchiarato)

___

## Hands-on:

- Precisamos instanciar nosso replica set. Você vai encontrar dois arquivos no
   repositório deste projeto (create_replset.sh e init.js).
   Com o terminal aberto execute primeiro o **create_replset.sh** e em seguida
   incie o seu replica set com o arquivo **init.js**:

{% highlight bash %}
$ sudo bash create_replset.sh
{% endhighlight %}

{% highlight bash %}
$ mongo localhost:27017 < init.js
{% endhighlight %}

- Instale a gem do [MongoDB Ruby Driver](https://github.com/mongodb/mongo-ruby-driver):

{% highlight ruby %}
gem 'mongo', '~> 2.0'
{% endhighlight %}

- Instancie um novo `client` informando os dados do *replica set*:

{% highlight ruby %}
client = Mongo::Client.new(['localhost:27017', 'localhost:27018', 'localhost:27019'], database: '[Informe aqui o nome do seu Database]', replica_set: 'replica_set')
{% endhighlight %}

- Para acessar uma collection do MongoDB, basta passá-la como parâmetro:

{% highlight ruby %}
client[:users].find.each do |user|
  puts "User name: #{ user[:name] }"
end
#=> prints 'User name: Foo' to STDOUT.
{% endhighlight %}

- Agora que podemos acessar os dados direto no MongoDB, vamos iterar em um loop
  infinito para podemors então matar um processo do mongo e ver como ele reage:

{% highlight ruby %}
loop do
  client[:users].find.each do |user|
    puts "Users name: #{ user }"
    puts '-' * 100
    sleep(3)
  end
end

#=> prints 'User name: Foo' to STDOUT.
#=> prints '--------------------------------------------------' to STDOUT.
#=> prints 'User name: Foo' to STDOUT.
#=> prints '--------------------------------------------------' to STDOUT.
#=> [...]
{% endhighlight %}

- Se você tentar matar o mongo nesse momento, o script vai falhar com uma
  exceção. Pois o drive deixa a critério do programador decidiro o que fazer.
  Desta forma eu criei uma forma de *retry* que trata a exceção lançada e tenta
  executar o processo mais uma vez:

{% highlight ruby %}
def failover
  begin
    yield if block_given?
  rescue Mongo::Error::SocketError, Errno::ECONNREFUSED => ex
    puts '- Server Down -'
    puts 'Retrying connections ...'
    sleep(2.5)
    puts '-' * 100
    retry
  end
end
{% endhighlight %}

- Agora com o *loop* adaptado, ele fica da seguinte forma:

{% highlight ruby %}
loop do
  failover do
    client[:users].find.each do |user|
      puts "Users name: #{ user }"
      puts '-' * 100
      sleep(3)
    end
  end
end
{% endhighlight %}

- Entre em qualquer instância do *replica set* e execute o comando rs.status()
  para descobrir qual é o nó primário:
{% highlight bash %}
$ mongo --port 27017
{% endhighlight %}

{% highlight bash %}
replica_set:SECONDARY>rs.status()
{% endhighlight %}

- Sabendo qual é a instância primária do seu *replica set*, logue na mesma e
  execute os comandos abaixo:

{% highlight bash %}
use admin
db.shutdownServer()
{% endhighlight %}

___

Após seguir todos os passos acima, você deve observar que durante algum tempo
após matar o processo primário, é logado no console a mensagem de 'Server Down'
mas logo em seguida, o nome do usuário volta a ser impresso na tela. Win!

![Failover em ação]({{ site.url  }}/assets/images/poc-01-failover.gif)
