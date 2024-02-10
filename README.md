# MagmaVizClient

This is a submodule of the [MagmaViz](https://github.com/MagmaMultiAgent/MagmaViz) repository, which is a visualization tool for Reinforcement Learning training.

The purpose of this is to provide a lightweight repository that you can download into your training container, so you don't have to clone the whole MagmaViz project just for sending data.

## Installation

Installing the MagmaVizClient is relatively straightforward.
1. Get the repository into your training container in your preferred way (download, clone, submodule, etc.)
2. Activate your Python virtual environment if you haven't already
3. Go into the `client` folder and run `pip install .`
4. You can now import the `Client` anywhere you want using the `from MagmaVizClient.client import Client` import statement


## Usage

Let's assume you already have a MagmaViz server running in your desired location (if you don't, I suggest visiting [this](https://github.com/MagmaMultiAgent/MagmaViz) repository).

To send data to that server you will need to set its IP address as an environmental variable. If the training and the MagmaViz server is running on the same system you might be able to just use `localhost`, however, if your training is running inside a container you might have to get the IP address of your host.

Windows instructions:
- You need [WSL2](https://learn.microsoft.com/en-us/windows/wsl/install) to run the MagmaViz server
- If you are training on the same WSL2 instance, you can use localhost
- If your training script is running inside a Docker for Windows container (with a WSL2 backend), it won't see your server on localhost
- In that case:
    1. Go to your [.wslconfig](https://learn.microsoft.com/en-us/windows/wsl/wsl-config) file and add `localhostforwarding=true` under `[wsl2]`
    2. Add `--network host` to your container run arguments
    3. If you still can't reach `localhost` from the container, you can try setting the IP of the WSL2 instance to the environmental variable
        1. Inside WSL2, type the following: `ip addr show eth0 | grep -oP '(?<=inet\s)\d+(\.\d+){3}'`
        2. Set the received IP address as the environmental variable `export VISUALIZATION_HOST={YOUR_WSL2_IP_ADDRESS}`


### Initializing the Client

After setting the MagmaViz host and importing the Client with `from MagmaVizClient.client import Client` you can initialize your client inside your training environment.
You can create more than one client.
Let's say I am using [Stable Baselines 3](https://stable-baselines3.readthedocs.io/en/master/) to train my model. I am interested in the state of my environment at every step, so I create a client inside my observation wrapper so that I can easily call it at every observation. If I am interested in plotting the reward, I create a client somewhere where I have access to that data.
Initializing the client doesn't require any arguments.


### Sending data

Sending the data can be done by calling the Client's `send` method with the data you want to visualize.
The data has to follow a strict format:

It has to be a list containing dictionaries. Each dictionary is a data sample for a step in an environment. If you want, you can create these samples batched and send multiple together, by appending more dictionaries to the list. But you can also call the `send` method at every step with a 1-length list, completely your preference. Keep in mind, that the data is sent to the server via UDP, so there is a packet limit you have to fit into. If you are not sure about the size of your data, you might want to consider sending small batches, or even 1 step at a time.

The dictionaries must contain the following properties:
```
{
	property_name: Required[str]
	env:           NotRequired[int]
	episode:       NotRequired[int]
	step:          Required[int]
	data:          Required[float | np.ndarray]
	file_name:     NotRequired[str]
}
```
The most important are the `property_name`, `step`, and `data` properties.
- `property_name` is the key under which your data will be stored in the backend. I suggest using a different property name for every different statistic (reward, board state, action count, etc.)
- `episode`, `step`, and `env` are used to order the received values.
    - `step` is a must, it refers to the index of the current step in the environment. You probably want to have some variable tracking the current step ID during training.
    - `episode` is the index of the current episode. It's not necessary, omitting it will cause it to be 0. You can simply use just the step as the time indicator, but if you want to group by the episode you have that option.
    - If you have multiple environments running at the same time please fill out the `env` property with the index of the environment, otherwise, the environments' data will get mixed.
- `data` is where you can put the data you want to display. Currently, the supported formats are scalar values, or numpy arrays, which are either 2 dimensional or 3 dimensional where the 3rd dimension's length is 3, to represent RGB values.
- `file_name` is optional. Every few steps the sent data is saved on the backend. The `file_name` indicates the name of the saved file. If you want to save your data to compare it later under a meaningful name, this is where you can put that name

The data is then pickled, compressed, and sent via UDP to the MagmaViz server where it can be displayed.
