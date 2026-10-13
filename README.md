1. Verify prerequisites
Run on the Linux host with Intel Gaudi hardware:
cat /etc/os-release
lspci -nn | grep -Ei 'habana|gaudi'
hl-smi
docker --version
docker compose version


Confirm that Gaudi is detected and the Docker installation works before continuing.
2. Clone the repository
mkdir -p ~/opea-workspace
cd ~/opea-workspace

git clone https://github.com/opea-project/GenAIExamples.git

cd GenAIExamples/FinanceAgent/docker_compose/intel/hpu/gaudi

pwd
ls -la


3. Set environment variables
Replace the example values with your actual API credentials.
export HF_TOKEN="YOUR_HUGGINGFACE_TOKEN"
export FINNHUB_API_KEY="YOUR_FINNHUB_API_KEY"
export FINANCIAL_DATASETS_API_KEY="YOUR_FINANCIAL_DATASETS_API_KEY"

export HF_CACHE_DIR="$PWD/data"
mkdir -p "$HF_CACHE_DIR"

export HOST_IP=$(hostname -I | awk '{print $1}')

source set_env.sh


Verify the configuration:
docker compose -f compose.yaml config -q
docker compose -f compose.yaml config --services


4. Pull images and start services
docker compose -f compose.yaml pull
docker compose -f compose.yaml up -d


Check container status:
docker compose -f compose.yaml ps


Check logs:
docker compose -f compose.yaml logs --tail=100


Check the Gaudi model server:
docker logs --tail=200 vllm-gaudi-server


5. Verify the LLM endpoint
curl -i http://127.0.0.1:8000/health


Expected: HTTP 200, provided port 8000 is published on the host.
6. Run the tests
Go to the FinanceAgent directory:
cd ~/opea-workspace/GenAIExamples/FinanceAgent
ls tests


Redis ingestion:
python tests/test_redis_finance.py \
  --port 6007 \
  --test_option ingest


Redis retrieval:
python tests/test_redis_finance.py \
  --port 6007 \
  --test_option get


FinQA agent:
export agent_port=9095
prompt="What is Gap's revenue in 2024?"

python3 tests/test.py \
  --prompt "$prompt" \
  --agent_role worker \
  --ext_port "$agent_port"


Research agent:
export agent_port=9096
prompt="generate NVDA financial research report"

python3 tests/test.py \
  --prompt "$prompt" \
  --agent_role worker \
  --ext_port "$agent_port" \
  --tool_choice get_current_date \
  --tool_choice get_share_performance


Supervisor agent:
export agent_port=9090

python3 tests/test.py \
  --agent_role supervisor \
  --ext_port "$agent_port" \
  --stream


Multi-turn test:
python3 tests/test.py \
  --agent_role supervisor \
  --ext_port "$agent_port" \
  --multi-turn \
  --stream


If Python dependencies are missing, install the dependencies specified by the repository's instructions before running these tests.
7. Access the UI
From a browser, open:
http://<HOST_IP>:5175


If it is inaccessible, check the port mappings:
cd ~/opea-workspace/GenAIExamples/FinanceAgent/docker_compose/intel/hpu/gaudi

docker compose -f compose.yaml ps
docker compose -f compose.yaml port agent-ui 5175


8. Troubleshoot
docker compose -f compose.yaml ps
docker compose -f compose.yaml logs --tail=200
docker logs --tail=200 vllm-gaudi-server
hl-smi
df -h
free -h


9. Stop the deployment
cd ~/opea-workspace/GenAIExamples/FinanceAgent/docker_compose/intel/hpu/gaudi

docker compose -f compose.yaml down
