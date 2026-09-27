# Subdomain Takeover Demo

This repository demonstrates a **Local Subdomain Takeover simulation**. It models how an attacker can reclaim a "dangling" internal reference when a legitimate sub-service is stopped or decommissioned, without anyone updating the core DNS configurations.

### ⚠️ Disclaimer

This project is built strictly for **educational purposes and local security research**. It provides a safe, contained sandbox environment to help you understand infrastructure asset management risks and how dangling pointers happen in the wild.

## 🛠️ Infrastructure Components

The project orchestrates three main nodes in an isolated bridge network (`172.20.0.0/16`):

- **`dns-server` (`172.20.0.2`)**: An internal BIND9 DNS server routing traffic for `app.company.local`.
    
- **`thirdparty-service` (`172.20.0.3`)**: The legitimate cloud or external service mapped to `thirdparty-service.local`.
    
- **`attacker` (`172.20.0.3`)**: A shadow container configured to emulate a hijacking attempt against `thirdparty-service.local`.
    

## ☣️ The Vulnerability: Dangling Pointers

The root zone configuration (`db.company.local`) sets up an internal canonical alias, mapping an official organizational subdomain directly to an external identifier:


```
app.company.local.      IN  CNAME   thirdparty-service.local.
```

If the physical target engine backing `thirdparty-service.local` drops off the network infrastructure while this DNS record remains active, the record becomes **dangling**. The organization still trusts `app.company.local`, but the destination address is completely unallocated and up for grabs.

## 🕹️ Execution Walkthrough

Follow this chronological sequence to reproduce the container-level takeover scenario step by step:

### 1. Initialize Safe Production Services

Spin up the DNS engine pointing to our third-party service:


```
docker compose up -d dns-server thirdparty-service
```

### 2. Verify DNS Registration

Check how `app.company.local` correctly points to `thirdparty-service.local` via the CNAME record:


```
dig @172.20.0.2 app.company.local
```

### 3. Check the Web Application

Inspect the website to see how the application looks and functions under normal conditions:


```
curl -s -H "Host: app.company.local" http://172.20.0.3 | html2text 
```

### 4. Decommission the Target Service

Simulate an infrastructure update, sudden failure, or migration where the third-party application is shut down, but the engineering team forgets to clean up the corresponding DNS configuration:


```
docker compose stop thirdparty-service
```

### 5. Observe the Dangling State

Notice how the web application is now offline, yet the DNS record is still dangling and pointing to the void:


```
curl -s -H "Host: app.company.local" http://172.20.0.3 | html2text 
dig @172.20.0.2 app.company.local
```

### 6. Deploy the Exploit

The attacker capitalizes on the abandoned resource by spinning up a rogue web application that claims the same `thirdparty-service.local` identifier/IP space:


```
docker compose --profile attack up -d attacker
```

### 7. Confirm the Takeover

Run the check again to see how the attacker successfully hijacked the domain name and is now serving malicious or unauthorized content under the trusted name:


```
curl -s -H "Host: app.company.local" http://172.20.0.3 | html2text 
```

## 🚮 Clean Up the Environment

If you want to run the demo again from scratch, make sure to completely clear out the environment to avoid stale network states or port conflicts:


```
docker compose --profile attack down --volumes --remove-orphans && docker system prune --networks -f
```

### Optional: Remove Images

If you want to wipe the downloaded container images as well:


```
docker rmi nginx:alpine ubuntu/bind9:latest
```
