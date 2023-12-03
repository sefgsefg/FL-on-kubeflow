# FL-on-kubeflow
A simple example of performing federated learning on kubeflow
#Part of the code comes from "https://github.com/stijani/tutorial?fbclid=IwAR2AvmE3DzXzuF6MxHuVUaP7_KLyOVIZK679d548jR2Gx4PlXKjZOU_DzuM"

## Description

Server
---
We use the "Flask" to make a server.
First create the list for storing client's upload data and make a dictionary for sharing variable between clients context

You should change "NUM_OF_CLIENTS" number for actual number of clients. This example is 2.
```
app = Flask(__name__)
    clients_local_count = []
    scaled_local_weight_list = []
    global_value = { #Share variable
                    'last_run_statue' : False, #last run finish or not
                    'data_statue' : None,      
                    'global_count' : None,     #global_count finish or not
                    'step' : None,
                    'weight_statue' : None,
                    'average_weights' : None,
                    'shutdown' : 0}
    
    NUM_OF_CLIENTS = 2 #number of clients
```
Create locks to ensure that the server calculation is correct
```
init_lock = threading.Lock()
clients_local_count_lock = threading.Lock()
scaled_local_weight_list_lock = threading.Lock()
cal_weight_lock = threading.Lock()
shutdown_lock = threading.Lock()
```
In the begining, the first enter client will lock and init the global variable, subsequent clients do not need to do it again.

Then the server will get client's data
```
@app.route('/data', methods=['POST'])
    def flask_server():
        with init_lock:  #check last run is finish and init variable
            
            while True:
                
                if(len(clients_local_count)==0 and global_value['last_run_statue'] == False):#init the variable by first client enter
                    global_value['last_run_statue'] = True
                    global_value['data_statue'] = False
                    global_value['scale_statue'] = False
                    global_value['weight_statue'] = False
                    break
                
                elif(global_value['last_run_statue'] == True):
                    break
                time.sleep(3)
        
        local_count = int(request.form.get('local_count'))          #get data
        bs = int(request.form.get('bs'))
        local_weight = json.loads(request.form.get('local_weight'))
        local_weight = [np.array(lst) for lst in local_weight]
```
Here is example for locking process.  The first enter client will detect the length of "clients_local_count" equal to NUM_OF_CLIENTS,
and set "global_value['data_statue'] " to True.  

The subsequent clients will enter "elif" part to prevent doing the same action.
```
with clients_local_count_lock:
            clients_local_count.append(int(local_count))
            
with scaled_local_weight_list_lock:
    while True:
        
        if (len(clients_local_count) == NUM_OF_CLIENTS and global_value['data_statue'] != True):
            global_value['last_run_statue'] = False
            sum_of_local_count=sum(clients_local_count)
            
            
            global_value['global_count'] = sum_of_local_count     
            
            scaling_factor=local_count/global_value['global_count']
            scaled_weights = scale_model_weights(local_weight, scaling_factor)
            scaled_local_weight_list.append(scaled_weights)
            
            global_value['scale_statue'] = True 
            global_value['data_statue'] = True
            break
        elif (global_value['data_statue'] == True and global_value['scale_statue'] == True):
            scaling_factor=local_count/global_value['global_count']
            scaled_weights =scale_model_weights(local_weight, scaling_factor)
            scaled_local_weight_list.append(scaled_weights)

            break
        time.sleep(1)
```
After finishing calculate the weight, the server has to clear the data to ensure the next FL round is coooect.
Then return the weight to clients.
```
        clients_local_count.clear()
        scaled_local_weight_list.clear()
        
        return jsonify({'result': (global_value['average_weights'])})
```
After the all FL rounds finish, the clients will post a signal to server.
When the number of signal equal to NUM_OF_CLIENTS, the server will shutdown.
```
@app.route('/shutdown', methods=['GET'])
def shutdown_server():
    global_value['shutdown'] +=1 
    with shutdown_lock:
        while True:
            if(global_value['shutdown'] == NUM_OF_CLIENTS):
                os._exit(0)
                return 'Server shutting down...'
            time.sleep(1)
```
