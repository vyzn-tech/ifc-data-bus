# start Docker
docker build -t nanomq .    
docker run -d --name nanomq -p 1883:1883 nanomq