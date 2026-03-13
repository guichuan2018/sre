**Design Specification For Secured Remote Command Execution**

**Design approach,**   
Worker Library: Core library that manages job lifecycle and resource control.  
API Server: Exposes the worker functionality over a secure gRPC interface. Node 2 node security is provided.  
CLI Client: Use pflag to provide a command line interface used to interact with the worker service. 

**Scope,**   
The following are included:  
Secure remote command execution  
Job lifecycle management  
Output streaming  
mTLS authentication  
Simple authorization model  
CPU, memory, and IO limits via cgroups  
CLI interface

**Proposed grpc APIs,** 

service RemoteExec {  
  rpc Start (StartRequest) returns (StartResponse);  
  rpc Stop (StopRequest) returns (StopResponse);  
  rpc GetStatus (StatusRequest) returns (StatusResponse);  
  rpc StreamOutput (OutputRequest) returns (stream OutputChunk);  
}

message StartRequest {  
  string command \= 1;  
  bool memory \= 2;  
  bool cpu \= 3;    
  bool disk\_io \= 4;

}

message StartResponse {  
  string id \= 1;  
}

message StopRequest {  
  string id \= 1;  
}

message StopResponse {  
  string message \= 1;  
}

message StatusRequest {  
  string id \= 1;  
}

message StatusResponse {  
  string status \= 1;  
}

message OutputRequest {  
  string id \= 1;  
}

message OutputChunk {  
  bytes data \= 1;  
}

By the way, a List API may be provided if time is available.

**Security considerations,**   
Authentication: with mTLS certificates, n2n(machine node to machine node) mutual authentication is provided.  

Authorization: the subject in certificates will be used as the identity for authorization.   
   
For the prototype, we will implement a simple "Role" check based on the certificate's Common Name (CN) or an OU field.

high level nodes: allows any actions, such as Start/Stop/Stream.  
low level nodes: allows Status/Stream only.

Note: this project is targeting node security. It does not provide authentication and authorization for the users who start the grpc calls? Shouldn’t the grpc server include it as well? Maybe, as it provides an even stronger security if necessity requires it.

**CLI:**  
**Client:**  
srec: secured remote execute client  
**% srec** \-a \<hostname\_or\_ip\> \-p \<port\_number\> \[-c \<location of my certificate\>\] \[-g \<control group\>\] **start**  
“command string” 

output:  
started, \<id\>

% **srec** \-a \<hostname\_or\_ip\> \-p \<port\_number\> \[-c \<location of my certificate\>\] \[-g \<control group\>\] **stop/get\_status/stream** \<id\>

output:  
for stop: stopped, \<id\>  
for get\_status: status, running, \<id\>  
for stream: output of the command

Additionally, the following subcommand may be provided if time is available.

// list all executed tasks   
**srec** \-a \<hostname\_or\_ip\> \-p \<port\_number\> \[-c \<location of my certificate\>\] \[-g \<control group\>\] **list**

//output:   
Root id, Command, Status  
123, ‘blahblah …’, Started  
125, ‘blahblah …’, Stopped  
126, ‘blahblah …’, Running

**Server:**  
**sres**: secured remote exec server

**sres** \-a \<hostname\_or\_ip\> \-p \<port\_number\> \[-c \<location of my certificate\>\] 

**mTLS:**  
   
Version: Minimum TLS 1.3.

Cipher Suites: TLS\_AES\_128\_GCM\_SHA256 or TLS\_CHACHA20\_POLY1305\_SHA256.

Authentication: Mutual TLS (mTLS). Both server and client must present certificates signed by a shared internal CA.

Client certificate must be signed by trusted CA and contain valid SAN

**implementation**

### **Output Streaming**

To support multiple concurrent readers and "start-from-beginning" streaming:

* Each job directs its output to a temporary file on disk.  
* StreamLogs opens the file, reads from the start, and push new bytes to the gRPC stream.  
* This avoids keeping massive byte slices in memory.