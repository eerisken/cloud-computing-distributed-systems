# **The Lean Cloud: Bare-Metal Wasm Orchestration**

This architecture eliminates the "Virtualization Tax." By running WebAssembly on bare metal, we achieve the security of a VM with the performance of a native binary and the density of thousands of instances per host.

## **1\. The Component: Rust PDF-Echo Server**

This server handles incoming text via JSON, logs the request metadata to a shared PostgreSQL instance, and generates a PDF.

### **Cargo.toml**

Ini, TOML

\[package\]  
name \= "pdf-echo-server"  
version \= "0.1.0"  
edition \= "2021"

\[dependencies\]  
warp \= "0.3"  
tokio \= { version \= "1", features \= \["full"\] }  
serde \= { version \= "1.0", features \= \["derive"\] }  
serde\_json \= "1.0"  
sqlx \= { version \= "0.7", features \= \["runtime-tokio", "postgres", "macros"\] }  
printpdf \= "0.7"  
uuid \= { version \= "1.0", features \= \["v4"\] }

### **src/main.rs**

Rust

use warp::Filter;  
use sqlx::postgres::PgPoolOptions;  
use printpdf::\*;  
use std::fs::File;  
use std::io::BufWriter;

\#\[tokio::main\]  
async fn main() {  
    let db\_url \= std::env::var("DATABASE\_URL").expect("DATABASE\_URL missing");  
      
    // Pool allows multiple simultaneous requests to share DB connections  
    let pool \= PgPoolOptions::new()  
        .max\_connections(20)  
        .connect(\&db\_url).await.expect("DB Connection failed");

    let echo\_route \= warp::post()  
        .and(warp::body::json())  
        .and(warp::addr::remote())  
        .and(warp::any().map(move || pool.clone()))  
        .and\_then(handle\_request);

    println\!("Server starting on port 8080...");  
    warp::serve(echo\_route).run((\[0, 0, 0, 0\], 8080)).await;  
}

async fn handle\_request(body: serde\_json::Value, addr: Option\<std::net::SocketAddr\>, pool: sqlx::PgPool) \-\> Result\<impl warp::Reply, warp::Rejection\> {  
    let text \= body\["text"\].as\_str().unwrap\_or("No Content");  
    let ip \= addr.map(|a| a.to\_string()).unwrap\_or\_else(|| "unknown".into());

    // Log to PostgreSQL  
    sqlx::query("INSERT INTO logs (ip, content) VALUES ($1, $2)")  
        .bind(\&ip).bind(text).execute(\&pool).await.ok();

    // PDF Generation  
    let (doc, page1, layer1) \= PdfDocument::new("Echo", Mm(210.0), Mm(297.0), "Layer 1");  
    let font \= doc.add\_builtin\_font(BuiltinFont::Helvetica).unwrap();  
    doc.get\_page(page1).get\_layer(layer1).use\_text(text, 12.0, Mm(10.0), Mm(280.0), \&font);  
      
    let filename \= format\!("output\_{}.pdf", uuid::Uuid::new\_v4());  
    let file \= File::create(\&filename).unwrap();  
    doc.save(\&mut BufWriter::new(file)).unwrap();

    Ok(warp::reply::json(\&serde\_json::json\!({"status": "pdf\_created", "file": filename})))  
}

## ---

**2\. The Database: PostgreSQL Schema**

Run this on your existing PostgreSQL instance to prepare the logging table.

SQL

CREATE DATABASE pdf\_cloud;  
\\c pdf\_cloud;

CREATE TABLE logs (  
    id SERIAL PRIMARY KEY,  
    ip VARCHAR(50),  
    content TEXT,  
    timestamp TIMESTAMP DEFAULT CURRENT\_TIMESTAMP  
);

## ---

**3\. The Infrastructure: Terraform Hardening**

Terraform will connect to your 5 Fedora servers from your laptop to install the runtime and harden the OS.

### **main.tf**

Terraform

variable "fedora\_ips" {  
  default \= \["10.0.0.1", "10.0.0.2", "10.0.0.3", "10.0.0.4", "10.0.0.5"\]  
}

resource "null\_resource" "hardening" {  
  count \= length(var.fedora\_ips)

  connection {  
    type     \= "ssh"  
    user     \= "fedora"  
    private\_key \= file("\~/.ssh/id\_rsa")  
    host     \= var.fedora\_ips\[count.index\]  
  }

  provisioner "remote-exec" {  
    inline \= \[  
      "sudo dnf install \-y wasmtime nomad consul",  
      "sudo systemctl enable \--now nomad consul",  
      \# Hardening: Firewall & SELinux  
      "sudo firewall-cmd \--permanent \--add-port=4646/tcp", \# Nomad  
      "sudo firewall-cmd \--permanent \--add-port=8080/tcp", \# App  
      "sudo firewall-cmd \--reload",  
      "sudo setsebool \-P nis\_enabled 1" \# Allow Wasmtime networking  
    \]  
  }  
}

## ---

**4\. The Orchestrator: Nomad Job**

This tells Nomad to run **6 instances** distributed across the cluster nodes.

### **pdf-echo.nomad**

Terraform

job "pdf-echo-service" {  
  datacenters \= \["dc1"\]

  group "workers" {  
    count \= 6 \# Total instances across 3-5 servers

    task "server" {  
      driver \= "wasm"  
      config {  
        path \= "local/pdf\_echo.wasm"  
        args \= \["--env", "DATABASE\_URL=postgres://user:pass@10.0.0.10:5432/pdf\_cloud"\]  
      }  
      artifact {  
        source \= "http://your-internal-repo/pdf\_echo.wasm"  
      }  
      resources {  
        cpu    \= 200  
        memory \= 64  
      }  
    }  
  }  
}

## ---

**5\. Execution: Commands to run on your Laptop**

Follow these steps to deploy your cloud from your workstation.

### **Step A: Build the Wasm Binary**

Bash

\# Add the Wasm target  
rustup target add wasm32-wasip1

\# Compile the project  
cargo build \--target wasm32-wasip1 \--release

\# The binary is now at: target/wasm32-wasip1/release/pdf\_echo\_server.wasm

### **Step B: Provision & Harden (Terraform)**

Bash

\# Initialize Terraform  
terraform init

\# Apply hardening to all 5 servers simultaneously  
terraform apply \-auto-approve

### **Step C: Deploy to Nomad**

Bash

\# Set your Nomad address (pick any of the 5 servers)  
export NOMAD\_ADDR="http://10.0.0.1:4646"

\# Run the job  
nomad job run pdf-echo.nomad

\# Check status  
nomad status pdf-echo-service

### **Step D: Monitor your Lean Cloud**

You can now open the **Nomad UI** by navigating to http://10.0.0.1:4646 in your browser. You will see 6 instances of your Rust PDF server running. Even if one Fedora server fails, Nomad will automatically restart the missing instances on the remaining healthy servers.  
---

**Would you like me to show you how to set up a "Load Balancer" (like Fabio or Traefik) so you can access all 6 servers through a single URL?**