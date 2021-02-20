FROM cm2network/steamcmd:latest

RUN /home/steam/steamcmd/steamcmd.sh +login anonymous +force_install_dir /home/steam/valheim +app_update 896660 validate +exit

WORKDIR /home/steam/valheim

# Prepare the persistence path as the current user so that volume mount doesn't wreck permissions on parent dirs
RUN mkdir -p /home/steam/.config/unity3d/IronGate/Valheim/worlds

EXPOSE 2456/udp
EXPOSE 2457/udp
EXPOSE 2458/udp

ENV VALHEIM_WORLD_NAME Dedicated

CMD LD_LIBRARY_PATH=./linux64:$LD_LIBRARY_PATH ./valheim_server.x86_64 -nographics -batchmode -name "${VALHEIM_SERVER_DISPLAY_NAME}" -port 2456 -world "${VALHEIM_WORLD_NAME}" -password "${VALHEIM_SERVER_PASSWORD}" -public 1