# FL-on-kubeflow
A simple example of performing federated learning on kubeflow
#Part of the code comes from "https://github.com/stijani/tutorial?fbclid=IwAR2AvmE3DzXzuF6MxHuVUaP7_KLyOVIZK679d548jR2Gx4PlXKjZOU_DzuM"

## Description
---
Server
---
```
app = Flask(__name__)
    clients_local_count = []
    scaled_local_weight_list = []
    global_value = { #Share variable
                    'last_run_statue' : False, #last run finish or not
                    'data_statue' : None,      #global_count finish or not
                    'global_count' : None,
                    'step' : None,
                    'weight_statue' : None,
                    'average_weights' : None,
                    'shutdown' : 0}
    
    NUM_OF_CLIENTS = 2 #number of clients
```
