
```bash
docker build -t stt-service:latest .
docker tag localhost/stt-service cluster-master:31234/stt-service:latest
docker push cluster-master:31234/stt-service:latest
```


