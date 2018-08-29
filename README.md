
  
# docker-mysql-replication    
    
> Caso necessário, mudar a permissão dos diretórios e subdiretórios master/ e slave/ para 0777 e os arquivos master/conf.d/master.cnf e slave/conf.d/slave.cnf para 0644    
 ## Iniciando os serviços    
1. Iniciar os serviços: `docker-compose up` 
2. Acessar máquinas com:   
   `docker exec -i -t mysql-master /bin/bash`   
 ou   
   `docker exec -i -t mysql-slave /bin/bash` 
3. Testar conexão entre máquinas com `ping mysql-master` na máquina correspondente ao slave e vice-versa.     
    
## Configuração da réplica    
### Master    
Confirmar os parâmetros do arquivo master/conf.d/master.cnf:    
``` 
bind-address            = 0.0.0.0 
server-id               = 1 # Cada máquina deve possuir um server-id único 
log_bin                 = /var/log/mysql/mysql-bin.log 
expire_logs_days        = 10 
max_binlog_size         = 100M 
binlog_do_db            = minha_base 
```    
Dentro do prompt do mysql:    
1. mysql > GRANT REPLICATION SLAVE ON *.* TO 'slave_user'@'%' IDENTIFIED BY 'password';     
2. mysql > FLUSH PRIVILEGES;    
3. mysql > SHOW MASTER STATUS;    
   > Salvar os valores para configurar no slave   

Dentro do shell:  
4. /# mysqldump -u root -p minha_base --routines --triggers --single-transaction --master-data=1 > /backup/backup.sql    
    
### Slave    
Confirmar os parâmetros do arquivo slave/conf.d/slave.cnf:    
``` 
server-id               = 2 
replicate-do-db         = minha_base 
``` 
Dentro do shell:  
5. /# mysql -u root -p minha_base < /backup/backup.sql    
6. /# ping mysql-master     
   > Anotar o IP da máquina para usar no comando 5 (substituir XXX.XXX.XXX.XXX)    
   
 Dentro do prompt do mysql:  
7. mysql >  CHANGE MASTER TO MASTER_HOST='XXX.XXX.XXX.XXX', MASTER_USER='slave_user',   MASTER_PASSWORD='password';  
8. mysql > START SLAVE;  
9. mysql > SHOW SLAVE STATUS;    
   > Verificar o log file e log pos com o comando `SHOW MASTER STATUS;` executado na etapa 3 (mysql-master).  
  
10. Caso necessário, executar os comandos   
    `STOP SLAVE;`  
    `CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.00000X', MASTER_LOG_POS=XXXX;`     
    `START SLAVE;`  
    Substituindo os valores de MASTER_LOG_FILE e MASTER_LOG_POS pelos encontrados na etapa 3.  
     
## Observações  
- Para ativar a réplica é necessário ter em mente a Engine do banco de réplica. Parar InnoDB não é necessário efetuar o lock das tabelas ao gerar o dump, basta usar a opção --single-transaction. Já para Engines não transacionais como MyISAM é necessário que seja efetuado o lock para garantir a consistência dos dados, fazendo com que algumas etapas sejam alteradas:

4. (4-mysql-master) Antes de executar o mysqldump é necessário abrir outro shell e acessar o prompt do mysql:

      4.1. mysql > USE minha_base;
      
      4.2. mysql > FLUSH TABLES WITH READ LOCK;
      
      4.3. mysql > SHOW MASTER STATUS;  
      > Anotar esse valor para usá-lo na configuração do slave.
      
      4.4 Dentro do primeiro shell, **com o outro ainda aberto**: /# mysqldump -u root -p minha_base > /backup/backup.sql
      
      4.5 Dentro do segundo shell: mysql > UNLOCK TABLES; 
      > O trabalho dentro desse shell foi concluído. Caso tenha anotado os valores do File e Position obtidos no comando 4.3, o mesmo pode ser fechado.
      

7. (7-mysql-slave) CHANGE MASTER TO MASTER_HOST='XXX.XXX.XXX.XXX', MASTER_USER='slave_user',   MASTER_PASSWORD='password', MASTER_LOG_FILE='mysql-bin.00000X', MASTER_LOG_POS=XXXX;
   > Substituir MASTER_HOST, MASTER_LOG_FILE e MASTER_LOG_POS pelos valores correspondentes