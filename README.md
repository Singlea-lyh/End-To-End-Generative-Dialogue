# End-to-End Generative Dialogue

A neural conversational model.

## Requirements

This code is written in Lua, and an installation of [Torch](https://github.com/torch/torch7/) is assumed. Training requires a few packages which can easily be installed through [LuaRocks](https://github.com/keplerproject/luarocks) (which comes with a Torch installation). Datasets are formatted and loaded using [hdf5](https://en.wikipedia.org/wiki/Hierarchical_Data_Format), which can be installed using this [guide](https://github.com/deepmind/torch-hdf5/blob/master/doc/usage.md).
```bash
$ luarocks install nn
$ luarocks install rnn
```
If you want to train on an Nvidia GPU using CUDA, you'll need to install the [CUDA Toolkit](https://developer.nvidia.com/cuda-toolkit) as well as the `cutorch` and `cunn` packages:
```bash
$ luarocks install cutorch
$ luarocks install cunn
```
If you'd like to chat with your trained model, you'll need the `penlight` package:
```bash
$ luarocks install penlight
```
If you would like to train your model in parallel you must install the parallel package and include our changes. 
```bash
$ PARALLEL_INSTALL=~/installs
$ PATH_TO_MAIN=~/Desktop/GoogleDrive/FinalProject/End-To-End-Generative-Dialogue
$ cd $PARALLEL_INSTALL
$ git clone https://github.com/clementfarabet/lua---parallel.git
$ cd lua---parallel
$ cp $PATH_TO_MAIN/stash/parallel/init.lua .
$ luarocks make
```

## Usage

### Data

Input data is stored in the `data` directory. At the moment, this code is only compatible with the MovieTriples dataset, as defined by [Serban et al., 2015](http://arxiv.org/abs/1507.04808). Unzip the MovieTriples dataset and place its contents into `data/MovieTriples`. *Note: the MovieTriples dataset is not publicly available, though training on arbitrary dialogue will be supported soon.*

**Preprocessing** is done using Python:
```bash
$ cd src
$ python preprocess.py
```
A limited dataset (limited input length) can also be used for testing:
```bash
$ python preprocess.py --seqlength 5
```

### Training

You can start training the model using `train.lua`.
```bash
$ th train.lua -data_file data/conv-train.hdf5 -val_data_file data/conv-val.hdf5 -save_file conv-model -gpuid 1
```
Here we are setting the flag `gpuid` to 1, which trains using the GPU. You can train on the CPU by omitting this flag or setting its value to -1. For a full list of model settings, please consult `$ th train.lua -help`.

**Checkpoints** are created after each epoch, and are saved within the `src` directory. Each checkpoint's name indicates the number of completed epochs of training as well as that checkpoint's [perplexity](https://en.wikipedia.org/wiki/Perplexity), essentially a measure of how confused the checkpoint is in its predictions. The lower the number, the better the checkpoint's predictions (and output text while sampling).

### Sampling

Given a checkpoint file, we can generate responses to input dialogue examples:
```bash
$ th run_beam.lua -model conv-model_epoch4.00_39.19.t7 -src_file data/dev_src_words.txt -targ_file data/dev_targ_words.txt -output_file pred.txt -src_dict data/src.dict -targ_dict data/targ.dict
```

### Chatting

It's also possible to chat directly with a checkpoint:
```bash
$ th chat.lua -model conv-model_epoch4.00_39.19.t7 -targ_dict data/targ.dict
```
These models have a tendency to respond tersely and vaguely. It's a work in progress!

## Advanced Usage

### Running code in parallel

**Local:** to run a worker with 4 parallel clients on your own machine:
```bash
$ th train.lua -data_file data/conv-train.hdf5 -val_data_file data/conv-val.hdf5 -save_file conv-model -parallel -n_proc 4
```

**Locally through localhost**: you can run a worker with 1 parallel client on your own computer through localhost (which is more similar to how things will work when running through a server). There is only 1 parallel client since it requires that you input your password while connecting to your own computer through ssh. This should not be used in practice as it's more inefficient than the previous command. This can, however, be a useful benchmark for developing remote server training.

First you must enable Remote Login in System Preferences > Sharing. You must also specify the location of the src folder from your home directory:
```bash
$ PATH_TO_SRC=Desktop/GoogleDrive/FinalProject/End-To-End-Generative-Dialogue/src/
$ PATH_TO_TORCH=/Users/michaelfarrell/torch/install/bin/th
$ th train.lua -data_file data/conv-train.hdf5 -val_data_file data/conv-val.hdf5 -save_file conv-model -parallel -n_proc 1 -localhost -extension $PATH_TO_SRC -torch_path $PATH_TO_TORCH

```
### Running with clients on Kevins computer
This is used as a comparison to the google servers (for debugging purposes). 
```bash
$ th train.lua -data_file data/conv-train.hdf5 -val_data_file data/conv-val.hdf5 -save_file conv-model -n_proc 1 -parallel -kevin -extension stash/mikeparallel/End-To-End-Generative-Dialogue/src/ -torch_path /Users/candokevin/torch/install/bin/th
```

### Running remotely on gcloud servers

**Set up an ssh key to connect to our servers:**
Replace USERNAME with your own username (i.e., USERNAME = michaelfarrell).
```bash
$ USERNAME=michaelfarrell
$ ssh-keygen -t rsa -f ~/.ssh/gcloud-sshkey -C $USERNAME
```
Hit enter twice and a key should have been generated. Now print the key and copy the output:
```bash
$ cat ~/.ssh/gcloud-sshkey.pub
```
Next you must add the key to the set of public keys:
- Login to our google compute account. 
- Go to compute engine dashboard
- Go to metdata tab
- Go to ssh-key subtab
- Click edit
- Add the key you copied as a new line

Restrict external access to the key:
```bash
$ chmod 400 ~/.ssh/gcloud-sshkey
```

**Generate an instance template:**
- Click on the 'Instance templates' tab
- Create new
- Name the template 
- Choose 8vCPU highmem as machine type
- Choose Ubuntu 14.04 LTS as boot disk
- Allow HTTP traffic
- Allow HTTPS traffic
- Under more->Disks, unclick 'Delete boot disk when instance is deleted'
- Create

**Allow tcp connections:**
- Click on the 'Instance templates' tab
- Click on the new template you created
- Go down to networks and click on the 'default' link
- Go to 'Firewall rules' and Add a new rule
- Set name to be 'all'
- Set source filter to allow from any source
- Under allowed protocols, put 'tcp:0-65535; udp:0-65535; icmp'
- Create


**Generate an instance group of machines if you have not yet done so:**
- Go to the "Instance groups" tab
- Create instance group
- Give the group a name, i.e. training-group-dev
- Give a description
- Set zone to us-central1-b
- Use instance template
- Choose 'miket=template' or other template of choice
- Set the number of instances
- Create
- Wait for the instances to launch
- Once there is a green checkmark, click on the new instance

**Connecting to GCloud Server**

You can connect to one of the servers by running:
```bash
$ IP_ADDR=130.211.160.115
$ ssh -o "StrictHostKeyChecking no" -i ~/.ssh/gcloud-sshkey $USERNAME@$IP_ADDR
```
where $username is the username you used to create the ssh key as defined above, and IP_ADDR is the ip address of the machine listed under "External ip" (i.e., 104.197.9.84). Note: the flag `-o "StrictHostKeyChecking no"` automatically adds the host to your list and does not prompt confirmation.

If you get an error like this:
```bash
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
```
then you'll want to
```bash
$ vim ~/.ssh/known_hosts
```
and delete the last few lines that were added. They should look like some ip address and then something that starts with AAAA. You can delete lines in vim by typing dd to delete the current line. This can happen when you restart the servers and they change ip addresses, among other things.

**Adding remote clients**
You will want to add your list of client servers to the file 'client_list.txt' where each line in the file is one of the external ip addresses located in the Instance group you are currently using. 

**Initializing remote servers**
Before using the remote servers, we need to make sure that the servers are ready to go. This can be done by running
```
$ python server_init.py
```
from the src folder on your own computer. 

**Running the remote server:** 
If the servers have been initialized, you will first want to connect to one of them:
```bash
$ ssh -o "StrictHostKeyChecking no" -i ~/.ssh/gcloud-sshkey $USERNAME@$IP_ADDR
```
Once connected, you need to again setup an ssh key as listed in the instructions: "Set up an ssh key to connect to our servers" above.
Once the key is created and added to the account, then:
```bash
$ cd End-To-End-Generative-Dialogue/src
$ th train.lua -data_file data/conv-train.hdf5 -val_data_file data/conv-val.hdf5 -save_file conv-model -parallel -n_proc -1 -remote -extension End-To-End-Generative-Dialogue/src/ -torch_path /home/michaelfarrell/torch/install/bin/th -n_proc 4

```

## Primary contributors

[Kevin Yang](https://github.com/kyang01)

[Michael Farrell](https://github.com/michaelfarrell76)

[Colton Gyulay](https://github.com/cgyulay)

## Relevant links

- https://medium.com/chat-bots/the-complete-beginner-s-guide-to-chatbots-8280b7b906ca#.u1jngyhzc
- https://www.youtube.com/watch?v=IK0t38Al4_E
- https://github.com/julianser/hed-dlg
- https://docs.google.com/document/d/1KKP8ZRZJbZweazZvz4cHZkvVnzFQApJJEySpZ5JLdwc/edit
- http://arxiv.org/pdf/1507.04808.pdf
- http://arxiv.org/pdf/1511.06931v6.pdf
- https://www.aclweb.org/anthology/P/P15/P15-1152.pdf
- http://arxiv.org/pdf/1603.09457v1.pdf
- https://www.reddit.com/r/datasets/comments/3bxlg7/i_have_every_publicly_available_reddit_comment/
- https://www.reddit.com/r/MachineLearning/comments/3ukvc6/datasets_of_one_to_one_conversations/
- http://arxiv.org/pdf/1412.3555v1.pdf
- https://github.com/clementfarabet/lua---parallel
- http://www.aclweb.org/anthology/P02-1040.pdf
- http://victor.chahuneau.fr/notes/2012/07/03/kenlm.html
- https://cloud.google.com/compute/docs/troubleshooting

## TODO

**Preprocessing (preprocess.py)**
- Add subTle datset cleaning to preprocessing code (and any other additional datasets we may need)
- Modify preprocessing code to have longer sequences (rather than just (U_1, U_2, U_3), have (U_1, ..., U_n) for some n. With this we could try to add more memory to the model we currently have now)
- Modify preprocessing code to return entire conversations (rather than fixing n, have the entire back and forth of a conversation together. This could be useful for trying to train a model more specific to our objective. This could be used for testing how the model does for a specific conversation )
- Finish cleaning up file (i.e. finish factoring code. I started this but things are going to be modified when subTle is added so I never finished. It shouldn't be bad at all)

**Parallel (parallel_functions.lua)**
- Add way to do localhost without password on server
- Get working on google servers
- Make sure server setup is correctly done

**General**
- Start result collection of some sort. Maybe have some datasheet and when we run a good model we record the results?
- Run each of the models for 10 epochs-ish? -> save the model, record results ^
- Experiment with HRED model
- Add word error rate when reporting

## Acknowledgments

Our implementation utilizes code from the following:

* [Yoon Kim's seq2seq-attn repo](https://github.com/harvardnlp/seq2seq-attn)
* [Element rnn library](https://github.com/Element-Research/rnn)
* [Facebook's neural attention model](https://github.com/facebook/NAMAS)
