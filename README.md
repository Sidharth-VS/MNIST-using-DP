# MNIST-using-DP

# 0. Clone Project Repository into Workspace

```bash
cd ~/swarm-learning/workspace/

git clone https://github.com/Sidharth-VS/MNIST-using-DP.git mnist
```

---

# 1. Generate Certificates

```bash
cd ~/swarm-learning/

cp -r examples/utils/gen-cert workspace/mnist/

./workspace/mnist/gen-cert -e mnist -i 1

./workspace/mnist/gen-cert -e mnist -i 2
```

---

# 2. Delete SWOP/SWCI Certificates

```bash
cd workspace/mnist/cert

rm swop-* swci-*

cd ../../../
```

---

# 3. Create Docker Network

```bash
docker network create host-1-net
```

---

# 4. Create Required Directories

```bash
mkdir -p ~/swarm-learning/workspace/mnist/tmp/sl1

mkdir -p ~/swarm-learning/workspace/mnist/tmp/sl2

mkdir -p ~/swarm-learning/workspace/mnist/results

chmod -R 777 ~/swarm-learning/workspace/mnist/tmp

chmod -R 777 ~/swarm-learning/workspace/mnist/results
```

---

# 5. Copy SwarmLearning Wheel

```bash
cp ~/swarm-learning/lib/swarmlearning-client-py3-none-manylinux_2_24_x86_64.whl \
~/swarm-learning/workspace/mnist/ml-context/swarmlearning-0.0.1-py3-none-manylinux_2_24_x86_64.whl
```

---

# 6. Build ML Docker Image

```bash
docker build -t mnist-ml-env \
~/swarm-learning/workspace/mnist/ml-context
```

---

# 7. Run APLS

```bash
docker run -d \
--name apls \
--network host-1-net \
-v apls-volume:/hpe \
-p 5814:5814 \
--restart unless-stopped \
hub.myenterpriselicense.hpe.com/hpe_eval/autopass/apls:9.19
```

---

# 8. Set Environment Variables

```bash
export HOST_IP=172.1.1.1

export SN_IP=172.1.1.1

export APLS_IP=172.1.1.1

export SN_API_PORT=30304
```

---

# 9. Run SN (Swarm Network Node)

```bash
cd ~/swarm-learning

./scripts/bin/run-sn -d --name=sn1 \
--network=host-1-net \
--host-ip=${HOST_IP} \
--sentinel \
--sn-api-port=${SN_API_PORT} \
--key=workspace/mnist/cert/sn-1-key.pem \
--cert=workspace/mnist/cert/sn-1-cert.pem \
--capath=workspace/mnist/cert/ca/capath \
--apls-ip=${APLS_IP}
```

---

# 10. Monitor SN Until Ready

```bash
docker logs -f sn1
```

Wait until:

```text
swarm.blCnt : INFO : Starting SWARM-API-SERVER on port: 30304
```

# Between Experiments

Stop old containers before every new experiment:

```bash
docker rm -f sn1 sl1 sl2 ml1 ml2 2>/dev/null
```

---

## Run SL1

```bash
./scripts/bin/run-sl -d --name=sl1 \
--network=host-1-net \
--host-ip=${HOST_IP} \
--sn-ip=${SN_IP} \
--sn-api-port=${SN_API_PORT} \
--sl-fs-port=16000 \
--key=workspace/mnist/cert/sl-1-key.pem \
--cert=workspace/mnist/cert/sl-1-cert.pem \
--capath=workspace/mnist/cert/ca/capath \
--ml-image=mnist-ml-env \
--ml-name=ml1 \
--ml-entrypoint=python3 \
--ml-cmd=/tmp/test/model/mnist.py \
-v ~/swarm-learning/workspace/mnist/tmp/sl1:/tmp/hpe-swarm \
--ml-v workspace/mnist/model:/tmp/test/model \
--ml-v workspace/mnist/results:/results \
--ml-e DATA_DIR=/app-data \
--ml-e SCRATCH_DIR=/tmp/scratch \
--ml-e RESULT_FILE=exp_weak_dp_adam_sl1.json \
--ml-e MIN_PEERS=2 \
--ml-e MAX_EPOCHS=8 \
--ml-e NODE_ID=0 \
--ml-e NUM_NODES=2 \
--ml-e DP_ENABLED=true \
--ml-e OPTIMIZER=adam \
--ml-e METRIC=both \
--ml-e NOISE_MULTIPLIER=1.0 \
--apls-ip=${APLS_IP}
```

```bash
docker logs -f ml1 > \
~/swarm-learning/workspace/mnist/results/exp_weak_dp_adam_ml1.log 2>&1 &
```

---

## Run SL2

```bash
./scripts/bin/run-sl -d --name=sl2 \
--network=host-1-net \
--host-ip=${HOST_IP} \
--sn-ip=${SN_IP} \
--sn-api-port=${SN_API_PORT} \
--sl-fs-port=17000 \
--key=workspace/mnist/cert/sl-2-key.pem \
--cert=workspace/mnist/cert/sl-2-cert.pem \
--capath=workspace/mnist/cert/ca/capath \
--ml-image=mnist-ml-env \
--ml-name=ml2 \
--ml-entrypoint=python3 \
--ml-cmd=/tmp/test/model/mnist.py \
-v ~/swarm-learning/workspace/mnist/tmp/sl2:/tmp/hpe-swarm \
--ml-v workspace/mnist/model:/tmp/test/model \
--ml-v workspace/mnist/results:/results \
--ml-e DATA_DIR=/app-data \
--ml-e SCRATCH_DIR=/tmp/scratch \
--ml-e RESULT_FILE=exp_weak_dp_adam_sl2.json \
--ml-e MIN_PEERS=2 \
--ml-e MAX_EPOCHS=8 \
--ml-e NODE_ID=1 \
--ml-e NUM_NODES=2 \
--ml-e DP_ENABLED=true \
--ml-e OPTIMIZER=adam \
--ml-e METRIC=both \
--ml-e NOISE_MULTIPLIER=1.0 \
--apls-ip=${APLS_IP}
```

```bash
docker logs -f ml2 > \
~/swarm-learning/workspace/mnist/results/exp_weak_dp_adam_ml2.log 2>&1 &
```

---