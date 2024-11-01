# Projeto Completo: Envio de Logs de Múltiplas Aplicações Node.js para o Graylog

Este projeto inclui a configuração completa de duas aplicações Node.js para envio de logs ao Graylog via Syslog, utilizando o `winston` e `winston-syslog`. Ambas as aplicações serão executadas em containers Docker e gerenciadas pelo PM2.

## Estrutura do Projeto

```
projeto-logs/
├── simple_app/
│   ├── Dockerfile
│   ├── ecosystem.config.js
│   ├── index.js
│   └── package.json
├── nova_app/
│   ├── Dockerfile
│   ├── ecosystem.config.js
│   ├── index.js
│   └── package.json
└── graylog/
    └── docker-compose.yml
```

## Aplicativo 1: `simple_app`

### Arquivo `package.json`
```json
{
  "name": "simple_app",
  "version": "1.0.0",
  "description": "Aplicacao Node.js simples para envio de logs ao Graylog",
  "main": "index.js",
  "scripts": {
    "start": "node index.js"
  },
  "dependencies": {
    "express": "^4.17.1",
    "winston": "^3.3.3",
    "winston-syslog": "^2.4.4"
  }
}
```

### Arquivo `index.js`
```javascript
const express = require('express');
const winston = require('winston');
require('winston-syslog');

const app = express();

// Configurando o logger para Syslog
const logger = winston.createLogger({
  level: 'debug',
  transports: [
    new winston.transports.Syslog({
      host: '192.168.0.103', // IP do servidor Graylog
      port: 514,             // Porta do Syslog
      protocol: 'udp4',      // Protocolo
      localhost: 'simple_app'
    })
  ]
});

// Endpoint raiz
app.get('/', (req, res) => {
  logger.info('Request recebido na rota raiz do simple_app');
  res.send('Hello, PM2 com Syslog!');
});

// Endpoint de exemplo para logs de erro
app.get('/error', (req, res) => {
  logger.error('Este é um log de erro de teste no simple_app');
  res.status(500).send('Simulação de erro no simple_app!');
});

// Inicia o servidor na porta 3000
const PORT = 3000;
app.listen(PORT, () => {
  logger.info(`Aplicacao simple_app rodando na porta ${PORT}`);
});
```

### Arquivo `ecosystem.config.js`
```javascript
module.exports = {
  apps: [
    {
      name: "simple_app",
      script: "./index.js",
      log_type: "syslog",
      out_file: "/dev/null",
      error_file: "/dev/null",
      merge_logs: true,
      env: {
        NODE_ENV: "production"
      }
    }
  ]
};
```

### Arquivo `Dockerfile`
```Dockerfile
FROM node:16
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm install && npm install pm2 -g
COPY . .
EXPOSE 3000
CMD ["pm2-runtime", "start", "ecosystem.config.js"]
```

## Aplicativo 2: `nova_app`

### Arquivo `package.json`
```json
{
  "name": "nova_app",
  "version": "1.0.0",
  "description": "Nova aplicacao Node.js para envio de logs ao Graylog",
  "main": "index.js",
  "scripts": {
    "start": "node index.js"
  },
  "dependencies": {
    "express": "^4.17.1",
    "winston": "^3.3.3",
    "winston-syslog": "^2.4.4"
  }
}
```

### Arquivo `index.js`
```javascript
const express = require('express');
const winston = require('winston');
require('winston-syslog');

const app = express();

// Configurando o logger para Syslog
const logger = winston.createLogger({
  level: 'debug',
  transports: [
    new winston.transports.Syslog({
      host: '192.168.0.103', // IP do servidor Graylog
      port: 514,             // Porta do Syslog
      protocol: 'udp4',      // Protocolo
      localhost: 'nova_app'
    })
  ]
});

// Endpoint raiz
app.get('/', (req, res) => {
  logger.info('Request recebido na rota raiz do nova_app');
  res.send('Hello, Graylog a partir do nova_app!');
});

// Endpoint de exemplo para logs de erro
app.get('/error', (req, res) => {
  logger.error('Este é um log de erro de teste no nova_app');
  res.status(500).send('Simulação de erro no nova_app!');
});

// Inicia o servidor na porta 3001
const PORT = 3001;
app.listen(PORT, () => {
  logger.info(`Aplicacao nova_app rodando na porta ${PORT}`);
});
```

### Arquivo `ecosystem.config.js`
```javascript
module.exports = {
  apps: [
    {
      name: "nova_app",
      script: "./index.js",
      log_type: "syslog",
      out_file: "/dev/null",
      error_file: "/dev/null",
      merge_logs: true,
      env: {
        NODE_ENV: "production"
      }
    }
  ]
};
```

### Arquivo `Dockerfile`
```Dockerfile
FROM node:16
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm install && npm install pm2 -g
COPY . .
EXPOSE 3001
CMD ["pm2-runtime", "start", "ecosystem.config.js"]
```

## Configuração do Graylog

### Arquivo `docker-compose.yml` para Subir o Graylog
```yaml
version: '3.8'

services:
  # MongoDB para armazenar os metadados do Graylog
  mongo:
    image: mongo:5.0
    container_name: graylog-mongo
    networks:
      - graylog_network

  # Elasticsearch para armazenar os logs recebidos
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.10.2
    container_name: graylog-elasticsearch
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"  # Ajuste a memória conforme necessário
    networks:
      - graylog_network

  # Graylog
  graylog:
    image: graylog/graylog:4.3
    container_name: graylog-server
    environment:
      - GRAYLOG_PASSWORD_SECRET=uma_senha_muito_secreta_e_longa
      - GRAYLOG_ROOT_PASSWORD_SHA2=<hash_da_senha>  # Substitua pelo hash SHA-256 da senha do admin
      - GRAYLOG_HTTP_EXTERNAL_URI=http://localhost:9000/
    depends_on:
      - mongo
      - elasticsearch
    ports:
      - "9000:9000"  # Porta para acessar a interface do Graylog
      - "514:514/udp"  # Porta Syslog UDP
      - "514:514/tcp"  # Porta Syslog TCP
    networks:
      - graylog_network

networks:
  graylog_network:
    driver: bridge
```

### Passos para Subir o Graylog

1. **Crie o arquivo `docker-compose.yml`** com o conteúdo acima.
2. **Suba os serviços do Graylog**:
   ```bash
   docker-compose up -d
   ```
3. **Acesse a interface do Graylog** no navegador em `http://localhost:9000`.
4. **Faça login** com o usuário `admin` e a senha configurada.
5. **Configure inputs de Syslog** para receber logs de ambas as aplicações.

## Como Executar

1. **Construa as
