
VERSION?=v1
DEPTH?=1.0
IMGSIZE?=224
MODEL_CKPT=mobilenet_$(VERSION)_$(DEPTH)_$(IMGSIZE)
MODEL_CKPT_TAR=$(MODEL_CKPT)_2017_06_14.tar.gz

# MobileNet model names as defined in https://github.com/tensorflow/models/blob/master/research/slim/nets/nets_factory.py
ifeq ($(DEPTH),1.0) 
MODEL_NAME=mobilenet_$(VERSION)
else ifeq ($(DEPTH),0.75)
MODEL_NAME=mobilenet_$(VERSION)_075
else ifeq ($(DEPTH),0.50)
MODEL_NAME=mobilenet_$(VERSION)_050
else ifeq ($(DEPTH),0.25)
MODEL_NAME=mobilenet_$(VERSION)_025
endif

OUTPUT_NODE_NAME=MobilenetV1/Predictions/Reshape_1

YELLOW='\033[1;33m'
NOCOLOR='\033[0m'



.PHONY: all
all: profile

PROJECT_DIR = $(CURDIR)
ZOO_RELATIVE_TF_SRC_PROJ = ../tf_src
ZOO_RELATIVE_TF_MODELS_PROJ = ../tf_models
ZOO_TF_SRC_PATH = ${PROJECT_DIR}/${ZOO_RELATIVE_TF_SRC_PROJ}/tensorflow
ZOO_TF_MODELS_PATH = ${PROJECT_DIR}/${ZOO_RELATIVE_TF_MODELS_PROJ}/models


# Checks if TF_MODELS_PATH is set.  If it isn't set then will make the
# tf_models project in the ncappzoo/tensorflow/tf_models directory.
check_model:
ifndef TF_MODELS_PATH
	@echo "TF_MODELS_PATH not set, making tf_models";
	(cd ../tf_models; make all; cd ${PROJECT_DIR})
	$(eval TF_MODELS_PATH = ${ZOO_TF_MODELS_PATH})
	@echo "TF_MODELS_PATH is ${TF_MODELS_PATH}" ;
else
	@echo "TF_MODELS_PATH is ${TF_MODELS_PATH}" ;
endif

# Checks if TF_SRC_PATH is set.  If it isn't set then will make the
# tf_models project in the ncappzoo/tensorflow/tf_src directory.
check_tf:
ifndef TF_SRC_PATH
	@echo "TF_SRC_PATH not set, making tf_src";
	(cd ../tf_src; make all; cd ${PROJECT_DIR})
	$(eval TF_SRC_PATH=${ZOO_TF_SRC_PATH})
	@echo "TF_SRC_PATH is ${TF_SRC_PATH}" ;
else
	@echo "TF_SRC_PATH is ${TF_SRC_PATH}" ;
endif

.PHONY: checkpoint
checkpoint:
	@echo $(YELLOW)"\nDownloading checkpoint files..."$(NOCOLOR)
	(mkdir -p model)
	(cd model; wget -nc http://download.tensorflow.org/models/$(MODEL_CKPT_TAR);)
	(cd model; tar -xvf $(MODEL_CKPT_TAR);)

.PHONY: export
export: check_model
	@echo $(YELLOW)"\nExporting GraphDef file..."$(NOCOLOR)
	(cd model; python3 $(TF_MODELS_PATH)/research/slim/export_inference_graph.py \
		--alsologtostderr \
		--model_name=$(MODEL_NAME) \
		--batch_size=1 \
		--dataset_name=imagenet \
		--image_size=$(IMGSIZE) \
		--output_file=$(MODEL_CKPT).pb;)

.PHONY: freeze
freeze: check_tf checkpoint export
	@echo $(YELLOW)"\nFreezing model for inference..."$(NOCOLOR)
	(cd model; python3 $(TF_SRC_PATH)/tensorflow/python/tools/freeze_graph.py \
		--input_graph=$(MODEL_CKPT).pb \
		--input_binary=true \
		--input_checkpoint=$(MODEL_CKPT).ckpt \
		--output_graph=$(MODEL_CKPT)_frozen.pb \
		--output_node_name=$(OUTPUT_NODE_NAME);)

.PHONY: compile
compile: freeze
	@echo $(YELLOW)"\nCompiling model to Movidius graph..."$(NOCOLOR)
	(cd model; mvNCCompile -s 12 $(MODEL_CKPT)_frozen.pb -in=input -on=$(OUTPUT_NODE_NAME);)
	@echo $(YELLOW)"Movidius graph generated! You can run inferences using ncappzoo/apps/image-classifier project.\n"$(NOCOLOR)

.PHONY: profile
profile: freeze
	@echo $(YELLOW)"\nProfiling the model..."$(NOCOLOR)
	(cd model; mvNCProfile -s 12 $(MODEL_CKPT)_frozen.pb -in=input -on=$(OUTPUT_NODE_NAME);)
	@echo $(YELLOW)"Movidius graph generated! You can run inferences using ncappzoo/apps/image-classifier project.\n"$(NOCOLOR)

.PHONY: check
check: freeze
	@echo $(YELLOW)"\nComparing results with standard Tensorflow..."$(NOCOLOR)
	(cd model; mvNCCheck -s 12 $(MODEL_CKPT)_frozen.pb -in=input -on=$(OUTPUT_NODE_NAME);)
	@echo $(YELLOW)"Movidius graph generated! You can run inferences using ncappzoo/apps/image-classifier project.\n"$(NOCOLOR)

.PHONY: run
run: compile
	@echo $(YELLOW)"\nRunning inferences using image-classifier project..."$(NOCOLOR)
	(python3 ../../apps/image-classifier/image-classifier.py --graph ./model/graph --dim $(IMGSIZE) $(IMGSIZE) --mean 127.5 --scale 0.00789 --colormode "RGB" --labels ./model/labels.txt --image ../../data/images/pic_053.jpg)

.PHONY: help
help:
	@echo $(YELLOW)"\nPossible make targets: "$(NOCOLOR);
	@echo "  make help - Shows this message.";
	@echo "  make - Builds all dependencies, but does not run this program.";
	@echo "  make checkpoint - Downloads pre-trained checkpoint files.;"
	@echo "  make export - Export the neural network model for inference.";
	@echo "  make freeze - Freeze the neural network model for inference.";
	@echo "  make compile - Convert the frozen model into Movidius graph file.";
	@echo "  make check - Compare inference results with that of TensorFlow running on CPU/GPU.";
	@echo "  make profile - Run the model on NCS and extract complexity, bandwidth and execution time for each layer.";

.PHONY: clean
clean:
	@echo $(YELLOW)"\nMaking clean..."$(NOCOLOR);
	rm -rf model
