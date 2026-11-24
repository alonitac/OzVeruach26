# Simple Storage Service (S3)

**Amazon S3** is AWS's storage for files. A file in S3 is an **object**. Objects live in a **bucket**. Each object has a **key**, which is its name inside the bucket.

New feature: Let's say a grower wants to upload a photo of a plot. We will use S3 to store the photo.


## Configure credentials

In this tutorial you'll talk to AWS from your local machine using **boto3**. boto3 is the official AWS SDK (Software Development Kit) for Python: a library that turns AWS API calls into Python functions. To talk to AWS on your behalf, boto3 needs credentials, so set them up first. Create an access key for your AWS user:

1. In the AWS console, click your account name (top right) → **Security credentials**.
2. Under **Access keys**, choose **Create access key**, then **Command Line Interface (CLI)**.
3. Keep the page open. You'll see the secret only once.

Install the [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) (Windows users: inside WSL) and run:

```bash
aws configure
```

Paste the access key and the secret. Region `us-east-1`, output `json`. This writes `~/.aws/credentials`, in your home folder, outside the repo. boto3 finds this file by itself, so you never pass credentials in code. Check it:

```bash
aws sts get-caller-identity
```

> [!IMPORTANT]
> An access key is a password. **NEVER** put it in `.env`, or in the code, or in a commit. Even if your coding agent tells you to, it's always a bad idea. The **ONLY** place to store it is your `~/.aws/credentials` file. If you find yourself copying it somewhere else, stop and think, you're doing something wrong.
> The only exception is in GitHub Actions, where you can store it in the repository secrets.


## Create a bucket

Let's first create a bucket.

1. Open the Amazon S3 console at [https://console.aws.amazon.com/s3/](https://console.aws.amazon.com/s3/). The region in the top right has to be `us-east-1`.
2. Choose **Create bucket**.
3. For **Bucket name**, enter `your-name-agropilot-photos`. The name is unique across all of S3, so use your own name. Lowercase letters, numbers, and hyphens.
4. For **Region**, choose `us-east-1`.
5. Leave the rest as it is, and choose **Create bucket**.

Now upload a photo by hand, so you can see what S3 stores.

1. Open the bucket.
2. Choose **Create folder**, name it `test`, and upload a photo into it.

The photo is now an object. Its key is the file name, for example `test/north.jpg`. Click it, then **Open**. That's the file you picked.

The console shows a folder structure, but S3 has no folders. Click the new object: its key is `test/north.jpg`. The console only splits keys on `/` to draw them like folders, but S3 stores bytes under a key, and that's all.


## The architecture

Obviously, we don't want growers to upload from the S3 console. The photo belongs to a plot, and plots live in `registry-service`. So the registry is the one that writes to S3:

```text
                                                          ┌──────────────────────┐
                                                   ┌────► │  database            │  plot row + photo key
 ┌─────────┐   ┌────────────┐   ┌──────────────────┴─┐    └──────────────────────┘
 │ Browser │ ─►│ Frontend   │ ─►│  registry-service  │
 └─────────┘   └────────────┘   └──────────────────┬─┘    ┌──────────────────────┐
                                                   └────► │  S3 bucket           │  {farm_id}/{plot_id}/photo
                                                          └──────────────────────┘
```


- **registry-service** checks that the plot exists, writes the bytes to S3, and saves the key in the database.

On the way back, `GET /v1/plots/{id}/photo` reads the key from the database, downloads the object from S3, and returns it.

The key in S3 will be `<farm_id>/<plot_id>/photo`. E.g.:

```text
b7e2c0a0-3c1d-4f0a-9a11-0c5d2e6f1001/c8f3d1b0-4d2e-5a1b-8b22-1d6e3f702002/photo
```

#### Why this key structure?

S3 is **object storage**, not a disk. It's not the regular disk your EC2 instance has, and it's not a file system. There are no real folders: a key like `farms/north/photo.jpg` is one flat string, and the `/` is just a character in it. You put a whole object, and you get a whole object, by its exact key.

One important implication of this is that you **can't edit files** or **search** a bucket by keyword, by content, or by the middle of a name. The only way to find objects without knowing their exact key is to **list by prefix**: "give me every key that starts with `farms/north/`". That one rule shapes how we design keys.

Since listing by prefix is the only way to find objects in S3, we design keys for the way we'll **read** them.

### boto3

To talk to S3 from Python we use **boto3**. These are the calls we'll need:

```python
s3 = boto3.client("s3", region_name="us-east-1")

s3.put_object(Bucket="your-name-agropilot-photos", Key="<farm_id>/<plot_id>/photo", Body=data, ContentType="image/jpeg")

obj = s3.get_object(Bucket="your-name-agropilot-photos", Key="<farm_id>/<plot_id>/photo")

page = s3.list_objects_v2(Bucket="your-name-agropilot-photos", Prefix="<farm_id>/")
```


## The feature

#### Step 1 - Start from `main`, create a branch

```bash
git checkout main
git pull
git checkout -b feat/plot-photo
```

#### Step 2 - Implement

Ask your coding agent to implement the feature:

> In `services/registry-service`, let a grower upload one photo for a plot.
>
> - `POST /v1/plots/{id}/photo` takes a multipart field `file` and returns `201` with `plot_id`, `key`, and `content_type`. The S3 key is `{farm_id}/{plot_id}/photo`, with the plot's current `farm_id`. A second upload overwrites it.
> - `GET /v1/plots/{id}/photo` downloads the object and returns it with its content type.
> - Unknown plot, or no photo yet: `404`, `code`: `NOT_FOUND`. Empty file, a content type other than `image/jpeg`, `image/png`, or `image/webp`, or a file over 5 MB: `422`, `code`: `VALIDATION`, and don't call S3. Errors are `application/problem+json`, like the other routes.
> - Any boto3 error (`botocore.exceptions.BotoCoreError` or `ClientError`, including missing credentials): log it, return `503`, `code`: `UNAVAILABLE`, and leave the stored key unchanged.
> - Store the key and the content type in the registry database. Don't add photo fields to the existing plot JSON.
> - Use boto3 (`uv add boto3`). Read `AWS_S3_BUCKET` and `AWS_REGION` in `config.py`, and add both to `.env.example`. Never pass credentials in code, let boto3 find them. Create the S3 client inside the request handler, not at import time.
>
> No tests yet. No frontend, no changes to `.github/workflows`.


#### Step 3 - Test it locally

Add the two settings to your local `services/registry-service/.env`:

```text
AWS_REGION=us-east-1
AWS_S3_BUCKET=your-name-agropilot-photos
```

To test your feature, you talk to the real S3 bucket from your local machine, using the credentials you configured at the top of this tutorial.

Now you can test your feature on your local machine. Use `curl` to upload a photo, make sure you see the object in the S3 bucket.

#### Step 4 - Push and open a PR

1. Commit the code, `uv.lock`, and the workflow change, and push the branch.
2. Open a pull request from `feat/plot-photo` into `main`.
3. Wait for **registry-service / test** to pass.

#### Step 6 - Deploy

Merge the PR. Let the pipeline deploy the new code.

You can test it from the instance:

```bash
curl -s -F "file=@north.jpg;type=image/jpeg" \
  http://localhost:8000/v1/plots/c8f3d1b0-4d2e-5a1b-8b22-1d6e3f702002/photo
```

It worked? No:

```json
{ "status": 503, "code": "UNAVAILABLE", ... }
```

The same code worked on your laptop. Let's look at the logs on the instance:

```bash
journalctl -u registry.service -n 50
```

```text
botocore.exceptions.NoCredentialsError: Unable to locate credentials
```

Remember how boto3 finds credentials. On your laptop, it found `~/.aws/credentials`, your access key. On the instance there is no such file, and that's on purpose: copying your personal key is a bad practice.


## Give the instance permission: IAM role

Who is allowed to do what in AWS is managed by **IAM** (Identity and Access Management). We'll cover IAM properly later. For now, we need one thing from it: a **role**.

Think of a role as a **mask** you put on an EC2 instance. The mask says what its wearer is allowed to do, for example "read and write objects in this bucket". When the instance wears the mask, AWS gives it short-lived credentials automatically, and boto3 picks them up by itself.

Let's create the role:

1. Open [IAM → Roles → Create role](https://console.aws.amazon.com/iam/home#/roles).
2. **Trusted entity type**: **AWS service**. **Use case**: **EC2**. Choose **Next**. This says who can wear the mask: EC2 instances.
3. Choose **Create policy**. A new tab opens. Choose **JSON**, and paste the policy below. Replace `YOUR-BUCKET` with your bucket name. Name it `agropilot-plot-photos` and create it.

    ```json
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": ["s3:PutObject", "s3:GetObject"],
          "Resource": "arn:aws:s3:::YOUR-BUCKET/*"
        }
      ]
    }
    ```

    This is what the mask allows: put and get objects in your bucket. Nothing else: no other bucket, no deleting, no listing.

4. Back in the role tab, refresh the list, select `agropilot-plot-photos`, and choose **Next**.
5. Name the role `agropilot-registry` and create it.

Now put the mask on: in the EC2 console, select your instance → **Actions** → **Security** → **Modify IAM role**. Choose `agropilot-registry` and **Update IAM role**.

Now test again

# Exercises

