Tutorial Karafka
================

Un tutorial basico de Karafka

prerequisitos
=============
tener docker (https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository)
y docker-compose instalados (https://docs.docker.com/compose/install/#install-compose-on-linux-systems)
tener ruby 2.6 preferiblemente con rvm


Crear carpeta de microservicios
```
$ mkdir microservicios
```

Crea un archivo docker-compose.yml con el siguiente contenido
```
version: '2'

networks:
  kafka-net:
    driver: bridge

services:
  zookeeper-server:
    image: 'bitnami/zookeeper:latest'
    networks:
      - kafka-net
    ports:
      - '2181:2181'
    environment:
      - ALLOW_ANONYMOUS_LOGIN=yes
  kafka-server1:
    image: 'bitnami/kafka:latest'
    networks:
      - kafka-net    
    ports:
      - '9092:9092'
    environment:
      - KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper-server:2181
      - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092
      - ALLOW_PLAINTEXT_LISTENER=yes
    depends_on:
      - zookeeper-server
```

Descargar las imagenes requeridas corriendo el siguiente comando en la carpeta donde creaste el archivo docker-compose.yml
```
$ docker-compose pull
```

Usar la version "correcta" de ruby
```
$ rvm use 2.6
```

Crear nuevo proyecto de Rails e instalar las gemas
```
$ rails new patients --api
$ cd patients
$ bundle install
```

Creamos el modelo de pacientes y su controlador
```
$ rails generate model Patient nombre:string apellido:string
$ rails db:migrate
$ rails generate controller Patients
```

Verificamos que su contenido quede así /app/controllers/patients_controller.rb
```
class PatientsController < ApplicationController
    def create
        patient = Patient.new(patient_params)
        if patient.save
            render :json => patient
        else
            render :json => patient.errors
        end
    end

    private
    def patient_params
        params.require(:patient).permit(:nombre, :apellido)
    end
end
```

y configuramos el ruteo a la raiz /config/routes.rb
```
Rails.application.routes.draw do
  # For details on the DSL available within this file, see https://guides.rubyonrails.org/routing.html
  resources :patients, :path => '/'
end
```

Iniciamos la aplicacion
```
$ rails server
```

Ejecutamos una prueba desde postman
```
POST localhost:3000/
{
    "nombre":"Arturo",
    "apellido":"Armendariz"
}
```

si en este momento tenemos un servicio que guarda un paciente, celebremos

------------------------------------------------------------------------------------

Hasta ahora todo "normal", es momento de emitir nuestros primeros eventos de kafka

detenemos nuestro rail server

Agregar la gema Karafka
agregar la linea ```gem 'karafka'``` al Gemfile
```
$ bundle install
```

Instalamos Karafka en nuestra aplicacion
```
$ karafka install
```

Instalar Kafka Tool
https://www.kafkatool.com/download.html

En ventana 2
	Iniciar Kafka << TARDA >>
	```
	$ docker-compose up
	```

Regresamos a ventana 1

creamos un responder /app/responders/patient_responder.rb
```
class PatientResponder < ApplicationResponder
    topic :patients
    def respond(patient)
        respond_to :patients, patient.to_json
    end
end
```

modificamos el /app/controllers/patients_controller.rb

```
    def create
        patient = Patient.new(patient_params)
        if patient.save
            PatientResponder.call(patient.to_json) # agregamos esta linea
            render :json => patient
        else
            render :json => patient.errors
        end
    end
```

Iniciamos nuestro rails server
```
$ rails server
```

Ejecutamos una prueba desde postman
```
POST localhost:3000/
{
    "nombre":"Benito",
    "apellido":"Benitez"
}
```

Revisamos con Kafka Tool y veremos que nuestro mensaje esta publicado.

si vemos un topico llamado "patients" y un evento de creacion de Benito Benitez, tenemos un microservicio de alta de pacientes, celebremos

------------------------------------------------------------------------------------

Y ahora, vamos a crear un servicio de mensajeria

Abrimos ventana 3
regresamos al directorio microservicios
```
$ cd microservicios
```

Crear nuevo proyecto de Rails
```
$ rails new messaging --api
$ cd messaging
```

Agregar la gema Karafka
agregar la linea ```gem 'karafka'``` al Gemfile
```
$ bundle install
```

Instalar Karafka
```
$ karafka install
```

Creamos un consumer /app/consumers/patient_consumer.rb
```
class PatientConsumer < ApplicationConsumer
    def consume
      params_batch.each do |message|
        Rails.logger.info "Enviando mensaje a: #{message.payload}"
      end
    end
end
```


registramos el consumer karafka.rb
```
  consumer_groups.draw do
    topic :patients do
      consumer PatientConsumer
    end
  ...
```

iniciamos el server de karafka (tambien debe estar corriendo el api de rails microservicios/patients> rails server)
```
$ karafka server
```

revisamos el log del servicio de messaging y debemos encontrar el mensaje impreso
```
[[example_app_patients] {patients: 0}:] Fetching batches
[[example_app_patients] {patients: 0}:] [fetch] Sending fetch API request 6 to localhost:9092
[[example_app_patients] {patients: 0}:] [fetch] Waiting for response 6 from localhost:9092
Params deserialization for patients topic successful in 1 ms
Enviando mensaje a: {"id":2,"nombre":"Benito","apellido":"Benitez","created_at":"2020-06-26T02:14:01.174Z","updated_at":"2020-06-26T02:14:01.174Z"}
Inline processing of topic patients with 1 messages took 3 ms
1 messages on patients topic delegated to PatientConsumer
[[example_app_patients] {}:] Marking patients/0:2 as processed
```

Si, ya procesamos mensajes en la cola, celebremos

------------------------------------------------------------------------------------

Es hora de publicar un evento al enviar el mensaje

detenemos karafka server con ctrl+c

creamos un responder /app/responders/message_responder.rb
```
class MessageResponder < ApplicationResponder
    topic :messages
    def respond(mensaje)
        respond_to :messages, "emisor: #{mensaje["emisor"]}, receptor: #{mensaje["receptor"]}, mensaje: #{mensaje["mensaje"]}"
    end
end
```

y notificamos cuando enviamos el mensaje /app/consumers/patient_consumer.rb
```
class PatientConsumer < ApplicationConsumer
    def consume
      params_batch.each do |message|
        Rails.logger.info "Enviando mensaje a: #{message.payload}"
        MessageResponder.call({"emisor"=>"yo","receptor"=>JSON.parse(message.payload)["nombre"],"mensaje"=>"Bienvenido!"}) #agregamos esta linea
      end
    end
end
```

iniciamos karafka server
```
$ karafka server
```

Ejecutamos una prueba desde postman
```
POST localhost:3000/
{
    "nombre":"Daniel",
    "apellido":"Dominguez"
}
```

revisamos los mensajes con Kafka Tool

si ahora tenemos el mensaje en el topico de patiens y un mensaje en el topico de messages, celebremos

------------------------------------------------------------------------------------

y la historia continua
