# ffn-tracer
This is a collection of techniques for neuron tracing based on flood-filling networks (FFN). 

The methods in this repository are primarily intended to adapt FFNs for use in neuron *tracing*, that is, mapping the "circuit" of the neuron as a set of nodes and edges (aka "skeletonization"). In particular, these methods are focused on training FFNs to do tracing on brightfield electron microscopy images (such as that shown below), which tend to be noisier and qualitatively different from the Serial Blockface Scanning Electron Microscopy (SBEM) and Focused Ion Beam Scanning Electron Microscopy (FIB-SEM) used in the original FFN applications.

The repository includes improved methods for training (such as regularization and adversarial training) as well as inference (including re-seeding methods which improve FFN tracing extension into long, thin fibers).


![example neuron](./img/patch_and_label_507727402_f32.png)

Before executing the steps below, set up and activate a virtual environment using the `requirements.txt` file.

In order to train a ffn-tracer model, follow these steps:

1. Determine seed locations for each neuron. Seed locations can either be manually determined by inspecting each image using a tool such as [GIMP](https://www.gimp.org/) or Vaa3D, or by a custom algorithm Seed locations should be stored in a CSV file. For an example, see `seed_locations.csv` in this repo.

2. Create `tfrecord`s and training data coordinates for your dataset. For example, to replicate the datasets used in our analysis, run:

    ``` 
    python generate_mozak_data.py \
        --dataset_ids 507727402 521693148 522442346 529751320 565040416 565298596 565636436 \
            565724110 570369389 319215569 397462955 476667707 476912429 495358721 508767079 \
            508821490 515548817 515843906 518298467 518358134 518784828 520260582 521693148 \
            521702225 522442346 541830986 548268538 550168314 \
        --gs_dir data/gold_standard \
        --img_dir data/img \
        --seed_csv data/seed_locations/seed_locations.csv \
        --out_dir ./data \
        --num_training_coords 5000 \
        --coord_margin 171
    ```
    
    This deposits a set of `tfrecord`s containing the training data into `out_dir/tfrecords`, one `tfrecord` per dataset, along with a set of `tfrecord`s containing coordinates to use for training examples (each coordinate contains the location of a ground-truth pixel).
    
3. *Training*:

    Set the learning rate and depth of the model. The default ffn learning rate is 0.001 and the depth is 9.

    ``` 
    export LEARNING_RATE=0.001
    export DEPTH=9
    export FOV=343
    ```

    a. Initiate model training. You should determine values for `image_mean` and `image_stddev` for your data. Set the desired number of training iterations via `max_steps`.
    
    ```
python train.py \
    --tfrecord_dir ./data${DATA}/tfrecords \
    --coordinate_dir ./data${DATA}/coords \
    --image_mean 78 --image_stddev 20 \
    --learning_rate $LEARNING_RATE \
    --optimizer $OPTIMIZER \
    --max_steps 10000000 \
    --model_args "{\"depth\": $DEPTH, \"fov_size\": [${FOV}, ${FOV}, 1], \"deltas\": [8, 8, 0], \"loss_name\": \"$LOSS\", \"alpha\": 1e-6"} \
    --visible_gpus=0,1
    ```
    
    b. (**optional, but recommended**) initiate TensorBoard to monitor training and view sample labeled images:
    
    `tensorboard --logdir ./training-logs`
    
    Note that if you are running training on a remote server, in order to view the TensorBoard output in your browser, you will first need to run `ssh -N -f -L localhost:16006:localhost:6006 user@hostname`.

4. *Inference*: Run inference on a new dataset.

  a. Generate the target volume as hdf5. Note that the working directory should contain a set of z-slice images as `.png` files. These are assembled into the test volume by `png_to_h5.py`. For more examples of input png data see e.g. the files [here](https://github.com/janelia-flyem/neuroproof_examples/tree/master/training_sample2/grayscale_maps).

  ```
  cd data/test/507727402
  python ../../../fftracer/utils/png_to_h5.py 507727402_raw.h5
  ```
  
  b. Set up the [Jupyter kernel](https://ipython.readthedocs.io/en/stable/install/kernel_install.html) if intending to use jupyter notebook for interence (recommended, since the notebook allows for manual seed specification and has an interactive visualization):
  
  ``` 
  source venv/bin/activate
  ipython kernel install --user --name=ffn
  ```
  
  After this, navigate to the location of jupyter kernels (on mac this is `/Users/yourname/Library/Jupyter/kernels/ffn`) and verify that the file `kernel.json` points to the correct PYthon interpreter (on some setups, the Python interpreter needs to be set manually). The `kernel.json` file should look like this:
  ```
      {
     "argv": [
      "path/to/repo/ffn-tracer/venv/bin/python", # line to check
      "-m",
      "ipykernel_launcher",
      "-f",
      "{connection_file}"
     ],
     "display_name": "ffn",
     "language": "python"
    }
  ```

  c. Run the inference step. The easiest way to do this is to run the interactive jupyter notebook that allows you to view the dynamic canvas as FFN inference proceeds.
  
  To run the inference notebook on a remote machine:
  
  On the remote machine, run
  ```
  jupyter notebook --no-browser --port=8889
  ```
  
  On the local machine, run
  ``` 
  ssh -N -f -L localhost:8888:localhost:8889 remote_user@remote_host
  ```
  
  Now open your browser on the local machine and type in the address bar [localhost:8888](localhost:8888).