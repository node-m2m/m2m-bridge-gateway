
## M2M Bridge Gateway
![](assets/m2m-gateway.svg)

In this example, an edge client will try to access an edge server from a local network on a different region. The communication path will change from a local network to a public internet traversing a new region and into the edge server local network.

All communication traffic from local network to public internet and vice versa are fully encrypted using TLS and a hybrid encryption - a combination of standard public and private encryption methods.  

<br>



#### 1. Create a project directory for each endpoint and install *m2m*.
#### 2. Create an app.js file inside each project directory and copy each code below one by one.
#### 3. Start each app.js application one by one.
```js
$ node app.js
```
### Edge Client
```js
const m2m = require('m2m')

let edge = new m2m.Edge()

m2m.connect(() => {
     let edgeClient = new edge.client({port:8140, restart:true})

     let pl = {sensor:true, type:'temperature', value:24}

     edgeClient.write('edge-data-source-1', pl, (data) => {
          console.log(data.toString())
     })

    edgeClient.on('error', (err) => {
         console.log('error', err.message)
     })
})
```
### M2M Bridge Client
```js
const m2m = require('m2m')
  
let m2mClient = new m2m.Client()
let edge = new m2m.Edge()

m2m.connect(() => {
    
    edge.createServer(8140, (server) => {
         server.dataSource('edge-data-source-1', (tcp) => {
             if(tcp.payload){
                 m2mClient.write(300, 'm2m-bridge-1', tcp.payload )
                 tcp.send('ack rcvd data')
             }
         })
    }) 
})
```
### M2M Bridge Server
```js
const m2m = require('m2m')  

let m2mServer = new m2m.Server(300)
let edge = new m2m.Edge()

m2m.connect(() => {
    let edgeClient = new edge.client({port:8150, restart:true})

    m2mServer.dataSource('m2m-bridge-1', (ws) => {
        if(ws.payload){
            edgeClient.write('edge-data-source-1', ws.payload)
        }
    })
})
```
### Edge Server
```js
const m2m = require('m2m')

let edge = new m2m.Edge()

m2m.connect(() => {
     edge.createServer(8150,  (server) => {
          server.dataSource('edge-data-source-1', (tcp) => {
              if(tcp.payload){
                 console.log('edge-data-source-1 rcvd data', tcp.payload)
                 tcp.end() 
              }
          })
     })
})
```
On the **edge client**, you should see a similar result as shown below.
```js
edge-data-source-1 rcvd data {sensor:true, type:'temperature', value:24}


```


