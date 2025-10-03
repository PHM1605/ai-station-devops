copy ~/.cloudflared/config.yml

## To create a tunnel on Cloudflared
```bash 
cloudflared tunnel create ai-station
```
Here we see the UUID of the tunnel

## To get credentials of Cloudflared
```bash
cloudflared tunnel token --cred-file ./creds.json ai-station
```

## to apply Cloudflared setup ONCE 
```bash 
sudo k3s kubectl apply -f infra/cloudflared/deploy.yaml
```