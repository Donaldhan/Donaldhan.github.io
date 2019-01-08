192.168.230.128:2181,192.168.230.128:2182,192.168.230.128:2183

./kafka-console-producer.sh --broker-list 192.168.230.128:9092 --topic my-replicated-topic  
./kafka-console-consumer.sh --bootstrap-server 192.168.230.128:9092 --from-beginning --topic my-replicated-topic 
