PWD=$(shell pwd)
DOCKER_IMAGE?=uscictdocker/mentor-pipeline:1.4.0-alpha.1
DOCKER_CONTAINER=mentor-pipeline
PROJECT_ROOT?=$(shell git rev-parse --show-toplevel 2> /dev/null)
AWS_REGION?=us-east-1
TRANSCRIBE_AWS_S3_BUCKET_SOURCE?=mentorpal-transcribe-source
TRANSCRIBE_MODULE_PATH?=transcribe_aws
DEV_ENABLED?=
DEV_ROOT?=$(shell cd ~/projects && pwd 2> /dev/null)
DEV_MENTOR_PIPELINE?=$(shell cd $(DEV_ROOT)/mentor-pipeline && pwd 2> /dev/null)
DEV_TRANSCRIBE?=$(shell cd $(DEV_ROOT)/py-transcribe && pwd 2> /dev/null)
DEV_TRANSCRIBE_AWS?=$(shell cd $(DEV_ROOT)/py-transcribe-aws && pwd 2> /dev/null)
DOCKER_PYTHON_VERSION=3.7
DOCKER_SITE_PACKAGES=/usr/local/lib/python$(DOCKER_PYTHON_VERSION)/site-packages
DOCKER_ENV_ARGS=\
	-e AWS_REGION=$(AWS_REGION) \
	-e AWS_ACCESS_KEY_ID=$(AWS_ACCESS_KEY_ID) \
	-e AWS_SECRET_ACCESS_KEY=$(AWS_SECRET_ACCESS_KEY) \
	-e TRANSCRIBE_AWS_S3_BUCKET_SOURCE=$(TRANSCRIBE_AWS_S3_BUCKET_SOURCE) \
	-e TRANSCRIBE_MODULE_PATH=$(TRANSCRIBE_MODULE_PATH)
DOCKER_VOLUME_ARGS_DATA_AND_VIDEO=\
	-v $(PWD)/data:/app/mounts/data \
	-v $(PWD)/videos:/app/mounts/videos
DOCKER_VOLUME_ARGS_DEV=
ifeq ("$(DEV_ENABLED)", "1")
ifneq ("$(DEV_TRANSCRIBE)", "")
	DOCKER_VOLUME_ARGS_DEV += -v $(DEV_TRANSCRIBE)/transcribe:$(DOCKER_SITE_PACKAGES)/transcribe
endif
ifneq ("$(DEV_TRANSCRIBE_AWS)", "")
	DOCKER_VOLUME_ARGS_DEV += -v $(DEV_TRANSCRIBE_AWS)/transcribe_aws:$(DOCKER_SITE_PACKAGES)/transcribe_aws
endif
ifneq ("$(DEV_MENTOR_PIPELINE)", "")
	DOCKER_VOLUME_ARGS_DEV += -v $(DEV_MENTOR_PIPELINE)/mentor_pipeline:/app/mentor_pipeline
endif
endif
DOCKER_ARGS=\
	$(DOCKER_ENV_ARGS) \
	$(DOCKER_VOLUME_ARGS_DATA_AND_VIDEO) \
	$(DOCKER_VOLUME_ARGS_DEV)
# virtualenv used for pytest
VENV=.venv
$(VENV):
	$(MAKE) venv-create

abs=$(shell d=$$(cd $$(dirname $(1)); pwd); f=$$(basename '$(1)') && echo "$${d}/$${f}")

a:
	@echo $(call abs,~/projects/mentor-pipeline/mentor_pipeline)

.PHONY: venv-create
venv-create: virtualenv-installed
	[ -d $(VENV) ] || virtualenv -p python3 $(VENV)
	$(VENV)/bin/pip install --upgrade pip
	$(VENV)/bin/pip install -r ./requirements.txt
	$(VENV)/bin/pip install -r ./requirements.test.txt

virtualenv-installed:
	$(PROJECT_ROOT)/bin/virtualenv_ensure_installed.sh

# Removes single mentor's data files from the local file system
.PHONY: data/mentors/%/clean
data/mentors/%/clean:
	@echo "cleaning data/mentors/$*/build..."
	@rm -rf "data/mentors/$*/build"

# Removes single mentor's data files from the local file system
.PHONY: videos/%/clean
videos/%/clean:
	@echo "cleaning videos/$*..."
	@rm -rf "videos/$*"

# Removes all mentor files from the local file system
.PHONY clean:
clean:
	@for m in data/mentors/*/*; do $(MAKE) data/mentors/$${m}/clean; done
	@for m in videos/*/*; do $(MAKE) videos/$${m}/clean; done

# Runs a shell inside the data processing pipeline dockerfile
.PHONY shell:
shell:
	docker run \
			-it \
			--rm \
			--name $(DOCKER_CONTAINER) \
			--entrypoint /bin/bash \
			$(DOCKER_ARGS) \
		$(DOCKER_IMAGE)


# Complete build of mentor data
# Runs build if necessary
# Generates data files
# TODO: 1) log every significant action (generating audio, transcribing), 2) build classifier for jd, 3) utterance yaml gets error codes, 4) make delete audio files that failed to transcribe
.PHONY: data/mentors-%
data/mentors-%:
	docker run \
			--rm \
			--name $(DOCKER_CONTAINER) \
			$(DOCKER_ARGS) \
			$(DOCKER_IMAGE) data-update --mentor $* --data=/app/mounts/data/mentors $(args)

.PHONY: data/topics_by_question.csv/mentor/%
data/topics_by_question.csv/mentors/%:
	docker run \
			--rm \
			--name $(DOCKER_CONTAINER) \
			-v $(PWD)/data:/app/mounts/data \
			$(DOCKER_IMAGE) topics-by-question-generate --mentor $* --data=/app/mounts/data/mentors


.PHONY: videos/mentors/%
videos/mentors/%: data/mentors/% 
	docker run \
			--rm \
			--name $(DOCKER_CONTAINER) \
			-v $(PWD)/data:/app/mounts/data \
			-v $(PWD)/videos:/app/mounts/videos \
			$(DOCKER_IMAGE) videos-update --mentor $* --data=/app/mounts/data/mentors

.PHONY: reduce-noise
reduce-noise:
	docker run \
			--rm \
			--name $(DOCKER_CONTAINER) \
			-v $(call abs,~/projects/mentor-pipeline/mentor_pipeline):/app/mentor_pipeline \
			-v $(call abs,/Users/kirschner/projects/mentor-pipeline/mentor_pipeline_runner.py):/app/mentor_pipeline_runner.py \
			-v $(call abs,$(noiseroot)):/app/mounts/noise \
			-v $(call abs,$(targetsroot)):/app/mounts/targets \
			$(DOCKER_IMAGE) reduce-noise --noise /app/mounts/noise/$(noise) /app/mounts/targets/$(targets)

.PHONY: reduce-noise-shell
reduce-noise-shell:
	docker run \
			--rm \
			-it \
			--name $(DOCKER_CONTAINER) \
			-v $(call abs,~/projects/mentor-pipeline/mentor_pipeline):/app/mentor_pipeline \
			-v $(call abs,/Users/kirschner/projects/mentor-pipeline/mentor_pipeline_runner.py):/app/mentor_pipeline_runner.py \
			-v $(call abs,$(noiseroot)):/app/mounts/noise \
			-v $(call abs,$(targetsroot)):/app/mounts/targets \
			--entrypoint /bin/bash \
			$(DOCKER_IMAGE)

# Build checkpoint from mentor data
.PHONY: checkpoint/%
checkpoint/%: data/mentors/%
	cd $(PROJECT_ROOT)/checkpoint && \
	CHECKPOINT=dev_latest $(MAKE) checkpoint-clean/mentor/$* checkpoint-train/mentor/$*
	@echo ""
	@echo "==== MAKE CHECKPOINT SUCCEEDED! ===="
	@echo ""
	@echo "If you have generated mentor videos, you can test the environment locally with:"
	@echo "	cd .. && make local-run-dev"
	@echo ""
	@echo "Then view your mentor here:"
	@echo "	http://localhost:8080/mentorpanel/?mentor=$*"
