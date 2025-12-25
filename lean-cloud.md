# **The Lean Cloud: Bare-Metal Wasm Orchestration**

This architecture eliminates the "Virtualization Tax." By running WebAssembly on bare metal, we achieve the security of a VM with the performance of a native binary and the density of thousands of instances per host.

## **1. The Component: Rust PDF-Echo Server**

This server handles incoming text via JSON, logs the request metadata to a shared PostgreSQL instance, and generates a PDF.

### **Cargo.toml**

```Ini, TOML

[package]  
name = "pdf-echo-server"  
version = "0.1.0"  
edition = "2021"

[dependencies]  
warp = "0.3"  
tokio = { version = "1", features = ["full"] }  
serde = { version = "1.0", features = ["derive"] }  
serde_json = "1.0"  
sqlx = { version = "0.7", features = ["runtime-tokio", "postgres", "macros"] }  
printpdf = "0.7"  
uuid = { version = "1.0", features = ["v4"] }
```

### **src/main.rs**

```Rust

use warp::Filter;  
use sqlx::postgres::PgPoolOptions;  
use printpdf::*;  
use std::fs::File;  
use std::io::BufWriter;

#[tokio::main]  
async fn main() {  
    let db_url = std::env::var("DATABASE_URL").expect("DATABASE_URL missing");  
      
    // Pool allows multiple simultaneous requests to share DB connections  
    let pool = PgPoolOptions::new()  
        .max_connections(20)  
        .connect(&db_url).await.expect("DB Connection failed");

    let echo_route = warp::post()  
        .and(warp::body::json())  
        .and(warp::addr::remote())  
        .and(warp::any().map(move || pool.clone()))  
        .and_then(handle_request);

    println!("Server starting on port 8080...");  
    warp::serve(echo_route).run(([0, 0, 0, 0], 8080)).await;  
}

async fn handle_request(body: serde_json::Value, addr: Option<std::net::SocketAddr>, pool: sqlx::PgPool) -> Result<impl warp::Reply, warp::Rejection> {  
    let text = body["text"].as_str().unwrap_or("No Content");  
    let ip = addr.map(|a| a.to_string()).unwrap_or_else(|| "unknown".into());

    // Log to PostgreSQL  
    sqlx::query("INSERT INTO logs (ip, content) VALUES ($1, $2)")  
        .bind(&ip).bind(text).execute(&pool).await.ok();

    // PDF Generation  
    let (doc, page1, layer1) = PdfDocument::new("Echo", Mm(210.0), Mm(297.0), "Layer 1");  
    let font = doc.add_builtin_font(BuiltinFont::Helvetica).unwrap();  
    doc.get_page(page1).get_layer(layer1).use_text(text, 12.0, Mm(10.0), Mm(280.0), &font);  
      
    let filename = format!("output_{}.pdf", uuid::Uuid::new_v4());  
    let file = File::create(&filename).unwrap();  
    doc.save(&mut BufWriter::new(file)).unwrap();

    Ok(warp::reply::json(&serde_json::json!({"status": "pdf_created", "file": filename})))  
}

```

**2. The Database: PostgreSQL Schema**

SQL Script to create the logging table on an existing PostgreSQL instance.

```SQL

CREATE DATABASE pdf_cloud;  
c pdf_cloud;

CREATE TABLE logs (  
    id SERIAL PRIMARY KEY,  
    ip VARCHAR(50),  
    content TEXT,  
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP  
);

```

**3. The Infrastructure: Terraform Hardening**

Terraform will connect to 5 Fedora servers from a laptop to install the runtime and harden the OS.

### **main.tf**

```Terraform

variable "fedora_ips" {  
  default = ["10.0.0.1", "10.0.0.2", "10.0.0.3", "10.0.0.4", "10.0.0.5"]  
}

resource "null_resource" "hardening" {  
  count = length(var.fedora_ips)

  connection {  
    type     = "ssh"  
    user     = "fedora"  
    private_key = file("~/.ssh/id_rsa")  
    host     = var.fedora_ips[count.index]  
  }

  provisioner "remote-exec" {  
    inline = [  
      "sudo dnf install -y wasmtime nomad consul",  
      "sudo systemctl enable --now nomad consul",  
      # Hardening: Firewall & SELinux  
      "sudo firewall-cmd --permanent --add-port=4646/tcp", # Nomad  
      "sudo firewall-cmd --permanent --add-port=8080/tcp", # App  
      "sudo firewall-cmd --reload",  
      "sudo setsebool -P nis_enabled 1" # Allow Wasmtime networking  
    ]  
  }  
}

```

**4. The Orchestrator: Nomad Job**

This tells Nomad to run **6 instances** distributed across the cluster nodes.

### **pdf-echo.nomad**

```Terraform

job "pdf-echo-service" {  
  datacenters = ["dc1"]

  group "workers" {  
    count = 6 # Total instances across 3-5 servers

    task "server" {  
      driver = "wasm"  
      config {  
        path = "local/pdf_echo.wasm"  
        args = ["--env", "DATABASE_URL=postgres://user:pass@10.0.0.10:5432/pdf_cloud"]  
      }  
      artifact {  
        source = "http://your-internal-repo/pdf_echo.wasm"  
      }  
      resources {  
        cpu    = 200  
        memory = 64  
      }  
    }  
  }  
}

```

**5. Execution: Commands to run on your Laptop**

Follow these steps to deploy the cloud from your workstation.

### **Step A: Build the Wasm Binary**

```Bash

# Add the Wasm target  
rustup target add wasm32-wasip1

# Compile the project  
cargo build --target wasm32-wasip1 --release

# The binary is now at: target/wasm32-wasip1/release/pdf_echo_server.wasm
```

### **Step B: Provision & Harden (Terraform)**

```Bash

# Initialize Terraform  
terraform init

# Apply hardening to all 5 servers simultaneously  
terraform apply -auto-approve
```

### **Step C: Deploy to Nomad**

```Bash

# Set your Nomad address (pick any of the 5 servers)  
export NOMAD_ADDR="http://10.0.0.1:4646"

# Run the job  
nomad job run pdf-echo.nomad

# Check status  
nomad status pdf-echo-service
```

### **Step D: Monitor your Lean Cloud**

Open the **Nomad UI** by navigating to http://10.0.0.1:4646 in a browser. You will see 6 instances of your Rust PDF server running. Even if one Fedora server fails, Nomad will automatically restart the missing instances on the remaining healthy servers.  
