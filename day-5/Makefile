# fraud-detection Makefile

setup:
	python3 -m venv mlops-venv && mlops-venv/bin/pip install -r requirements.txt

data:
    python3 src/data/process_data.py

train:
	python3 src/models/train.py

test:
	pytest tests/

clean:
	rm -rf __pycache__

all: setup train test
