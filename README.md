
## M2M Bridge Gateway
![](assets/m2m-gateway.png)

In this example, an edge client from New York will try to access an edge server from Tokyo. The communication path will traverse a local network from New York through the public internet entering a new region then through a local network in Tokyo to access the edge server.

All communication traffic from local network to public internet and vice versa are fully encrypted using TLS and a combination of standard public and private encryption methods.  

<br>



#### 1. Create a project directory for each endpoint and install *m2m*.
#### 2. Create an app.js file inside each project directory and copy each code below correspondingly to each endpoints.
#### 3. Start each app.js application one by one.
```js
$ node app.js
```
### Edge Client
```js
const m2m = require('m2m')

let edge = new m2m.Edge({name:'edge client'})

let main = async () => {
     await m2m.connect()
     
     /*********************
     
          Edge client Section
     
     *********************/
     let edgeClient = new edge.client({port:8140, restart:true})
     
     edgeClient.on('ready', (result) => {
          console.log('edge server 8140 ready', result) // should be true if up and false if down
     })  
     
     edgeClient.on('error', (err) => {
          console.log('edge client error', err.message)
     })
     
     edgeClient.write('edge-data-source-1', 'sensor-1', (data) => {
          console.log('write', data)
     })

     edgeClient.subscribe('edge-publish-data-1', (data) => {
          console.log('sub', data)
          if(data.value < 30){
               edgeClient.write('edge-data-source-1', 'sensor-2')
          }
          else if(data.value > 105){
               edgeClient.write('edge-data-source-1', 'sensor-1')
          } 
     })
}

main()
```
### M2M Client Bridge
```js
const m2m = require('m2m')
  
let client = new m2m.Client({name:'m2m client bridge'})
let edge = new m2m.Edge({name:'edge server'})

let currentValue = ''

let main = async () => {
     await m2m.connect()

     /*********************
     
       M2M Client Section
     
     *********************/
     let m2mClient = new client.access(300)
     
     m2mClient.subscribe('m2m-bridge-2', (data) => {
          currentValue = data
     })  

     /*********************
     
       Edge Server Section
     
     *********************/
     const edgeServer = edge.createServer(8140)

     edgeServer.dataSource('edge-data-source-1', async (tcp) => {
          let result = ''
          // write 
          if(tcp.payload){
               result = await m2mClient.write('m2m-bridge-1', tcp.payload )
          }
          // read
          else{
               result = await m2mClient.read('m2m-bridge-1')
          }
          tcp.send(result)   
     })
     
     edgeServer.publish('edge-publish-data-1', async (tcp) => {
          tcp.send(currentValue)  
     })  
}

main()
```
### M2M Server Bridge
```js
const m2m = require('m2m')  

let m2mServer = new m2m.Server(300)
let edge = new m2m.Edge({name:'edge client'})

let main = async () => {
     await m2m.connect()
     
     /*********************
     
       Edge client Section
     
      *********************/
     let edgeClient = new edge.client({port:8150, restart:true})
     
     edgeClient.on('ready', (result) => {
          console.log('edge server 8150 ready', result) // should be true if up and false if down
     })
     
     edgeClient.on('error', (err) => {
          console.log('edge client error', err.message)
     })    
     
     /*********************
     
       M2M Server Section
     
     *********************/
     m2mServer.dataSource('m2m-bridge-1', async (ws) => {
          let result = ''
          // write
          if(ws.payload){
               result = await edgeClient.write('edge-data-source-1', ws.payload)
          }
          // read
          else {
               result = await edgeClient.read('edge-data-source-1')
          }
          ws.send(result)
     })
     
     m2mServer.publish('m2m-bridge-2', async (ws) => {
          let result = await edgeClient.read('edge-data-source-1')
          ws.send(result)
     })
}

main()
```
### Edge Server
```js
const m2m = require('m2m')

let edge = new m2m.Edge({name:'edge server'})

function sensor1(){
  return 25 + Math.floor(Math.random() * 10)
}

function sensor2(){
  return 100 + Math.floor(Math.random() * 10)
}

let currentSensor = 'sensor-1';

let main = async () => {
     await m2m.connect()
     
     /*********************
     
      Edge Server Section
     
     *********************/
     const edgeServer = edge.createServer(8150)
     
     edgeServer.dataSource('edge-data-source-1', (tcp) => {
          // write
          if(tcp.payload){
               currentSensor = tcp.payload
               tcp.send({topic:tcp.topic, currentSensor:currentSensor})       
          }
          // read
          else{
               if(currentSensor === 'sensor-1'){
                    tcp.send({topic:tcp.topic, sensor:currentSensor, value:sensor1()}) 
               }
               else if(currentSensor === 'sensor-2'){
                    tcp.send({topic:tcp.topic, sensor:currentSensor, value:sensor2()}) 
               }
               else{
                    tcp.send({topic:tcp.topic, result:'invalid sensor'}) 
               }
          }
     })
}

main()
```
On the **edge client**, you should see a similar result as shown below.
```js
edge server 8140 ready true
sub { topic: 'edge-data-source-1', sensor: 'sensor-1', value: 32 }
write { topic: 'edge-data-source-1', currentSensor: 'sensor-1' }
sub { topic: 'edge-data-source-1', sensor: 'sensor-1', value: 27 }
sub { topic: 'edge-data-source-1', sensor: 'sensor-2', value: 101 }
...
```


