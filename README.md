# tch-rs
Rust bindings for PyTorch. The goal of the `tch` crate is to provide some thin wrappers
around the C++ PyTorch api (a.k.a. libtorch). It aims at staying as close as
possible to the original C++ api. More idiomatic rust bindings could then be
developed on top of this. The [documentation](https://docs.rs/tch/) can be found on docs.rs.

[![Build Status](https://travis-ci.org/LaurentMazare/tch-rs.svg?branch=master)](https://travis-ci.org/LaurentMazare/tch-rs)
[![Latest version](https://img.shields.io/crates/v/tch.svg)](https://crates.io/crates/tch)
[![Documentation](https://docs.rs/tch/badge.svg)](https://docs.rs/tch)
![License](https://img.shields.io/crates/l/tch.svg)


The code generation part for the C api on top of libtorch comes from
[ocaml-torch](https://github.com/LaurentMazare/ocaml-torch).

## Getting Started

This crate requires the C++ PyTorch library (libtorch) in version *v1.1.0* to be available on
your system. You can either install it manually and let the build script know about
it via the `LIBTORCH` environment variable. If not set, the build script will
try downloading and extracting a pre-built binary version of libtorch.

### Libtorch Manual Install

- Get `libtorch` from the
[PyTorch website download section](https://pytorch.org/get-started/locally/) and extract
the content of the zip file.
- Add the following to your `.bashrc` or equivalent, where `/path/to/libtorch` is the
path to the directory that was created when unzipping the file.
```bash
export LIBTORCH=/path/to/libtorch
export LD_LIBRARY_PATH=${LIBTORCH}/lib:$LD_LIBRARY_PATH
```
- You should now be able to run some examples, e.g. `cargo run --example basics`.

## Examples

### Basic Tensor Operations

This crate provides a tensor type which wraps PyTorch tensors. Here is a minimal
example of how to perform some tensor operations.

```rust
extern crate tch;
use tch::Tensor;

fn main() {
    let t = Tensor::of_slice(&[3, 1, 4, 1, 5]);
    let t = t * 2;
    t.print();
}
```

### Writing a Simple Neural Network

The `nn` api can be used to create neural network architectures, e.g. the following code defines
a simple model with one hidden layer and trains it on the MNIST dataset using the Adam optimizer.

```rust
extern crate tch;
use tch::{nn, nn::Module, nn::OptimizerConfig, Device};

const IMAGE_DIM: i64 = 784;
const HIDDEN_NODES: i64 = 128;
const LABELS: i64 = 10;

fn net(vs: &nn::Path) -> impl Module {
    nn::seq()
        .add(nn::linear(vs / "layer1", IMAGE_DIM, HIDDEN_NODES, Default::default()))
        .add_fn(|xs| xs.relu())
        .add(nn::linear(vs, HIDDEN_NODES, LABELS, Default::default()))
}

pub fn run() -> failure::Fallible<()> {
    let m = tch::vision::mnist::load_dir("data")?;
    let vs = nn::VarStore::new(Device::Cpu);
    let net = net(&vs.root());
    let opt = nn::Adam::default().build(&vs, 1e-3)?;
    for epoch in 1..200 {
        let loss = net
            .forward(&m.train_images)
            .cross_entropy_for_logits(&m.train_labels);
        opt.backward_step(&loss);
        let test_accuracy = net
            .forward(&m.test_images)
            .accuracy_for_logits(&m.test_labels);
        println!(
            "epoch: {:4} train loss: {:8.5} test acc: {:5.2}%",
            epoch,
            f64::from(&loss),
            100. * f64::from(&test_accuracy),
        );
    }
    Ok(())
}
```

More details on the training loop can be found in the
[detailed tutorial](https://github.com/LaurentMazare/tch-rs/tree/master/examples/mnist).

### Using some Pre-Trained Model

The [pretrained-models  example](https://github.com/LaurentMazare/tch-rs/tree/master/examples/pretrained-models/main.rs)
illustrates how to use some pre-trained computer vision model on an image.
The weights - which have been extracted from the PyTorch implementation - can be
downloaded here [resnet18.ot](https://github.com/LaurentMazare/ocaml-torch/releases/download/v0.1-unstable/resnet18.ot)
and here [resnet34.ot](https://github.com/LaurentMazare/ocaml-torch/releases/download/v0.1-unstable/resnet34.ot).

The example can then be run via the following command:
```bash
cargo run --example pretrained-models -- resnet18.ot tiger.jpg
```
This should print the top 5 imagenet categories for the image. The code for this example is pretty simple.

```rust
    // First the image is loaded and resized to 224x224.
    let image = imagenet::load_image_and_resize(image_file)?;

    // A variable store is created to hold the model parameters.
    let vs = tch::nn::VarStore::new(tch::Device::Cpu);

    // Then the model is built on this variable store, and the weights are loaded.
    let resnet18 = tch::vision::resnet::resnet18(vs.root(), imagenet::CLASS_COUNT);
    vs.load(weight_file)?;

    // Apply the forward pass of the model to get the logits and convert them
    // to probabilities via a softmax.
    let output = resnet18
        .forward_t(&image.unsqueeze(0), /*train=*/ false)
        .softmax(-1);

    // Finally print the top 5 categories and their associated probabilities.
    for (probability, class) in imagenet::top(&output, 5).iter() {
        println!("{:50} {:5.2}%", class, 100.0 * probability)
    }
```


Further examples include:
* A simplified version of
  [char-rnn](https://github.com/LaurentMazare/tch-rs/blob/master/examples/char-rnn)
  illustrating character level language modeling using Recurrent Neural Networks.
* [Neural style transfer](https://github.com/LaurentMazare/tch-rs/blob/master/examples/neural-style-transfer)
  uses a pre-trained VGG-16 model to compose an image in the style of another image (pre-trained weights:
  [vgg16.ot](https://github.com/LaurentMazare/ocaml-torch/releases/download/v0.1-unstable/vgg16.ot)).
* Some [ResNet examples on CIFAR-10](https://github.com/LaurentMazare/tch-rs/tree/master/examples/cifar).
* A [tutorial](https://github.com/LaurentMazare/tch-rs/tree/master/examples/jit)
  showing how to deploy/run some Python trained models using
  [TorchScript JIT](https://pytorch.org/docs/stable/jit.html).
* Some [Reinforcement Learning](https://github.com/LaurentMazare/tch-rs/blob/master/examples/reinforcement-learning)
  examples using the [OpenAI Gym](https://github.com/openai/gym) environment. This includes a policy gradient
  example as well as an A2C implementation that can run on Atari games.
* A [Transfer Learning Tutorial](https://github.com/LaurentMazare/tch-rs/blob/master/examples/transfer-learning)
  shows how to finetune a pre-trained ResNet model on a very small dataset.

## License
`tch-rs` is distributed under the terms of both the MIT license
and the Apache license (version 2.0), at your option.

See [LICENSE-APACHE](LICENSE-APACHE), [LICENSE-MIT](LICENSE-MIT) for more
details.
