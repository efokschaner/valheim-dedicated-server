# Valheim Dedicated Server on GCP using Docker GCE Instance

## Container runtime environment vars
**Required**:
- `VALHEIM_SERVER_DISPLAY_NAME`: The name shown in the servers listing.
- `VALHEIM_SERVER_PASSWORD`: The pw players will need to connect.

**Optional**:
- `VALHEIM_WORLD_NAME`: defaults to "Dedicated". This selects the .fwl and .db world files which will be used.

## Notes
The setup assumes you are running Windows locally, and uses powershell syntax, but the commands themselves should be largely portable.

The world files are mounted from the compute instance to survive container-restart (from `/mnt/stateful_partition/valheim-worlds`).

The server may not show up in the in-game server browser. This is still under investigation.

## Build
```
docker build -t valheim-server ./docker
```

## Run locally
We'll use a local folder called `persistent-worlds` to store the world data across server restarts. The world will be created on first use. You can also add additional world files OR replace the files `Dedicated.fwl` and `Dedicated.db` with any existing world files you want.

*Note: World files do not seem to be able to be renamed without breaking them, therefore use the `VALHEIM_WORLD_NAME` env var if your world was not already named `"Dedicated"`.*

```
docker run -it -v "%cd%/persistent-worlds:/home/steam/.config/unity3d/IronGate/Valheim/worlds" -p 127.0.0.1:2456-2458:2456-2458/udp -e VALHEIM_SERVER_DISPLAY_NAME=AServerHasNoName -e VALHEIM_SERVER_PASSWORD=password valheim-server
```
Test by having Valheim client "Join IP" 127.0.0.1.

## Deploy to GCP
- Ensure you have your own build of the container by doing the build described above.
- Replace `valheim-001` in the following commands with your desired GCE instance name.

### First time
```
docker tag valheim-server us.gcr.io/$(gcloud config get-value project)/valheim-server
docker push us.gcr.io/$(gcloud config get-value project)/valheim-server

# In the following command, specify VALHEIM_SERVER_DISPLAY_NAME, VALHEIM_SERVER_PASSWORD, and VALHEIM_WORLD_NAME as desired
gcloud compute instances create-with-container valheim-001 `
    --container-image us.gcr.io/$(gcloud config get-value project)/valheim-server:latest `
    --machine-type e2-highcpu-4 `
    --boot-disk-size 10GB `
    --zone us-west1-b `
    --network-tier STANDARD `
    --tags valheim-server `
    --container-env VALHEIM_SERVER_DISPLAY_NAME=<...> `
    --container-env VALHEIM_SERVER_PASSWORD=<...> `
    --container-env VALHEIM_WORLD_NAME=b<...> `
    --container-mount-host-path 'mount-path=/home/steam/.config/unity3d/IronGate/Valheim/worlds,host-path=/mnt/stateful_partition/valheim-worlds,mode=rw' `
    --metadata-from-file=user-data=valheim-server.cloud-config.yaml

gcloud compute firewall-rules create allow-valheim `
    --direction ingress `
    --action allow `
    --target-tags valheim-server `
    --rules udp:2456-2458
```

### Upload a world

#### Stop the server to avoid conflict with files in use
```
gcloud compute ssh valheim-001
# Get the container id for valheim-server
docker ps
# Kill the id for the valheim-server container
docker kill <container id>
```
#### Upload an existing world with scp
```
# As the chronos user to match the way gcp container-os writes the files from the container
gcloud compute scp "C:\Users\<YOUR_USER>\AppData\LocalLow\IronGate\Valheim\worlds\<YOUR_WORLD_NAME>.*" chronos@valheim-001:/mnt/stateful_partition/valheim-worlds
```

### Restart instance to restart container runner
```
gcloud compute ssh valheim-001
sudo shutdown -r now
```

### Redeploy
```
docker tag valheim-server us.gcr.io/just-a-shell/valheim-server
docker push us.gcr.io/just-a-shell/valheim-server
# Reboot the target instance to trigger update of container
gcloud compute ssh valheim-001 --command='sudo shutdown -r now'
```