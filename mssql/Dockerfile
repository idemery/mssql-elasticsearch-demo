FROM mcr.microsoft.com/mssql/server:2019-GA-ubuntu-16.04
ENV SA_PASSWORD "qweQWE123"
ENV ACCEPT_EULA "Y"

USER root 

ADD --chown=mssql:root https://github.com/Microsoft/sql-server-samples/releases/download/wide-world-importers-v1.0/WideWorldImporters-Full.bak /var/opt/mssql/backup/wwi.bak

USER mssql

RUN /opt/mssql/bin/sqlservr --accept-eula & (echo "awaiting mssql bootup for 15 seconds" && sleep 15 && echo "restoring.."  && /opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P 'qweQWE123' -Q 'RESTORE DATABASE WideWorldImporters FROM DISK = "/var/opt/mssql/backup/wwi.bak" WITH MOVE "WWI_Primary" TO "/var/opt/mssql/data/WideWorldImporters.mdf", MOVE "WWI_UserData" TO "/var/opt/mssql/data/WideWorldImporters_userdata.ndf", MOVE "WWI_Log" TO "/var/opt/mssql/data/WideWorldImporters.ldf", MOVE "WWI_InMemory_Data_1" TO "/var/opt/mssql/data/WideWorldImporters_InMemory_Data_1"')
