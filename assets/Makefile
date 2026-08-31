# ==============================================================================
# Dependencies (must be installed before using this Makefile):
#
#   System:
#     - make          : this tool
#     - zip           : for packaging the modules folder  (apt install zip)
#     - java          : required by Spark                 (apt install default-jdk)
#     - Apache Spark  : spark-submit must be on PATH
#     - Apache Hadoop : hdfs must be on PATH (only needed for hdfs-upload/run-hdfs)
#
#   Python (inside the venv):
#     - venv-pack     : for packaging the venv            (.venv/bin/pip install venv-pack)
#     - pyspark       : must match the Spark installation  (listed in requirements.txt)
#
#   Setup steps:
#     1. python3 -m venv .venv
#     2. .venv/bin/pip install -r requirements.txt        (includes venv-pack)
#     3. Verify: which spark-submit && which hdfs
# ==============================================================================

# ==============================================================================
# Config — edit these to match your setup
# ==============================================================================
APP_NAME        := iris_etl
MAIN            := main.py
MODULES_DIR     := modules
VENV_DIR        := .venv

# Optional: set to a .jar file or a comma-separated list of .jar files.
# Leave empty if no JARs are needed.
JARS            :=
# JARS          := /home/smag/py2_etl/mariadb-java-client-3.5.8.jar

MASTER          := spark://localhost:7077
DEPLOY_MODE     := client
NUM_EXECUTORS   := 2
EXECUTOR_MEMORY := 1g
DRIVER_MEMORY   := 1g

HDFS_USER_DIR   := /user/$(shell whoami)

# CLI args forwarded to main.py (override on command line)
ARGS ?=

# ==============================================================================
# Derived — don't edit below this line
# ==============================================================================
VENV_ZIP        := venv.tar.gz
VENV_PACK       := $(shell which venv-pack 2>/dev/null || echo $(VENV_DIR)/bin/venv-pack)
MODULES_ZIP     := modules.zip

# Driver (local) uses the real venv; executors use the unpacked archive
PYSPARK_PYTHON        := $(VENV_DIR)/bin/python   # driver  — real local path
PYSPARK_PYTHON_WORKER := ./venv/bin/python         # executors — unpacked archive path

# Build --jars flag only if JARS is set
JARS_FLAG       := $(if $(JARS),--jars $(JARS),)


.PHONY: all run install package clean hdfs-upload run-hdfs help

# ------------------------------------------------------------------------------
all: help

# ------------------------------------------------------------------------------
## Install/update dependencies from requirements.txt into the venv
install:
	$(VENV_DIR)/bin/pip install -r requirements.txt

# ------------------------------------------------------------------------------
## Run the job (packages first, then submits)
run: package
	PYSPARK_PYTHON=$(PYSPARK_PYTHON) \
	PYSPARK_PYTHON_WORKER=$(PYSPARK_PYTHON_WORKER) \
	spark-submit \
		--master $(MASTER) \
		--deploy-mode $(DEPLOY_MODE) \
		--name $(APP_NAME) \
		--num-executors $(NUM_EXECUTORS) \
		--executor-memory $(EXECUTOR_MEMORY) \
		--driver-memory $(DRIVER_MEMORY) \
		$(JARS_FLAG) \
		--archives $(VENV_ZIP)#venv \
		--py-files $(MODULES_ZIP) \
		$(MAIN) $(ARGS)

# ------------------------------------------------------------------------------
## Package venv and modules into archives for distribution
package: $(VENV_ZIP) $(MODULES_ZIP)

# Rebuild venv.tar.gz if requirements.txt has changed
$(VENV_ZIP): requirements.txt
	@echo ">>> Installing dependencies into virtualenv..."
	pip install -r requirements.txt   # install first
	@echo ">>> Packaging virtualenv with venv-pack..."
	$(VENV_PACK) -f -p $(VENV_DIR) -o $(VENV_ZIP)
	@echo "    $(VENV_ZIP) ready"

# Rebuild modules.zip if any .py file in modules/ changes
$(MODULES_ZIP): $(shell find $(MODULES_DIR) -name '*.py')
	@echo ">>> Packaging modules..."
	cd $(MODULES_DIR) && zip -qr ../$(MODULES_ZIP) .
	@echo "    $(MODULES_ZIP) ready"

# ------------------------------------------------------------------------------
## Upload packages to HDFS (optional — speeds up repeated runs)
hdfs-upload: package
	hdfs dfs -mkdir -p $(HDFS_USER_DIR)/$(APP_NAME)
	hdfs dfs -put -f $(VENV_ZIP)    $(HDFS_USER_DIR)/$(APP_NAME)/
	hdfs dfs -put -f $(MODULES_ZIP) $(HDFS_USER_DIR)/$(APP_NAME)/
	$(if $(JARS),hdfs dfs -put -f $(JARS) $(HDFS_USER_DIR)/$(APP_NAME)/,)
	@echo ">>> Uploaded to HDFS: $(HDFS_USER_DIR)/$(APP_NAME)/"

## Run using files already on HDFS (run hdfs-upload first)
run-hdfs: package hdfs-upload
	PYSPARK_PYTHON=$(PYSPARK_PYTHON) \
	PYSPARK_PYTHON_WORKER=$(PYSPARK_PYTHON_WORKER) \
	spark-submit \
		--master $(MASTER) \
		--deploy-mode $(DEPLOY_MODE) \
		--name $(APP_NAME) \
		--num-executors $(NUM_EXECUTORS) \
		--executor-memory $(EXECUTOR_MEMORY) \
		--driver-memory $(DRIVER_MEMORY) \
		$(if $(JARS),--jars $(HDFS_USER_DIR)/$(APP_NAME)/$(notdir $(JARS)),) \
		--archives     $(HDFS_USER_DIR)/$(APP_NAME)/$(VENV_ZIP)#venv \
		--py-files     $(HDFS_USER_DIR)/$(APP_NAME)/$(MODULES_ZIP) \
		$(MAIN) $(ARGS)

# 		--py-files     $(HDFS_USER_DIR)/$(APP_NAME)/$(MODULES_ZIP) \

# ------------------------------------------------------------------------------
## Remove generated archives
clean:
	rm -f $(VENV_ZIP) $(MODULES_ZIP)
	@echo ">>> Cleaned up archives"

# ------------------------------------------------------------------------------
## Show this help
help:
	@echo ""
	@echo "Usage: make <target>"
	@echo ""
	@grep -E '^##' Makefile | sed 's/^## /  /'
	@echo ""
