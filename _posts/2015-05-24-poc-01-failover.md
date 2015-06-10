---
layout: post
title: "#1 PoC: Failover com Ruby e MongoDB"
description: "Veja na prática como o MongoDB se recupera de falhas (failover)"
categories:
- Ruby
- MongoDB
tags: [ruby, mongodb]
permalink: failover-com-ruby-e-mongodb
comments: true
---

Quando vi pela primeira vez como funciona o sistema de recuperação de falhas do
MongoDB (**Replica Set**) foi como se um mundo novo tivesse se aberto.
Veja com esse POC, como funciona na prática o processo de substituição  de nós e
recuperação de falhas do MongoDB.

## O que é Failover e Replica Set?

Failover é a forma com que um sistema pode se recuperar de uma falha sendo
substituído por um sistema secundário.

Você pode saber mais, acessando o post que eu escrevi para o Awesome blog [Ship
It](http://shipit.resultadosdigitais.com.br/blog/alta-disponibilidade-e-tolerancia-a-falhas-com-mongodb/) da [Resultados Digitais](http://www.resultadosdigitais.com.br).

___

Neste PoC, criei um script Ruby que realiza um loop infinito lendo
os dados de um documento do MongoDB.

Todo o código que usei pode ser encontrado no meu
[Github](https://github.com/tchiarato/Failover)

___

## Hands-on:

- Primeiro precisamos instanciar nosso replica set. Você vai encontrar dois arquivos no
   repositório deste projeto (create_replset.sh e init.js).
   No terminal execute os comandos abaixo:

{% highlight bash %}
$ sudo bash create_replset.sh
{% endhighlight %}

{% highlight bash %}
$ mongo < init.js
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

- Se você tentar matar o mongo nesse momento o script vai falhar com uma
  exceção. O drive deixa a critério do programador o tratamento das exceções
  lançadas.
  Desta forma eu criei um *retry* manual que trata a exceção lançada e tenta
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

- Com o *loop* adaptado, ele fica da seguinte forma:

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

- Agora você pode matar a instância primária do seu replica set para ver o
  failover em ação. Use os comandos abaixo para matar o processo:

{% highlight bash %}
# Esteja logado na instância primária do seu replica set
> use admin
> db.shutdownServer()
{% endhighlight %}

- Para descobrir qual é o nó primário do replica set, execute os comandos abaixo:

{% highlight bash %}
$ mongo --port 27017
{% endhighlight %}

{% highlight bash %}
# Instância localhost:27017
> rs.status()
{% endhighlight %}

___

Após seguir todos os passos acima você deve observar que após matar o processo primário,
durante algum tempo é exibido no console a mensagem 'Server Down', porém, logo em seguida o nome do usuário volta a ser impresso na tela. Win!

<script type="text/javascript" src="https://asciinema.org/a/21239.js" id="asciicast-21239" async data-autoplay="true" data-loop="true"></script>
