---
layout: tutorial_hands_on

title: "Use Singularity containers for running Galaxy jobs"
zenodo_link: ""
questions:
objectives:
  - Configure your Galaxy to use Singularity and BioContainers for running jobs
time_estimation: "1h"
key_points:
contributors:
  - torfinnnome
  - mvdbeek
  - bernt-matthias
  - hexylena
subtopic: features
tags:
  - jobs
requirements:
  - type: "internal"
    topic_name: admin
    tutorials:
      - ansible
      - ansible-galaxy
---

# Overview
{:.no_toc}

In this tutorial you will learn how to configure Galaxy to run jobs using [Singularity](https://sylabs.io/singularity/) containers provided by the [BioContainers](https://biocontainers.pro/) community.

## Background

> BioContainers is a community-driven project that provides the infrastructure and basic guidelines to create, manage and distribute bioinformatics packages (e.g conda) and containers (e.g docker, singularity). BioContainers is based on the popular frameworks Conda, Docker and Singularity.
>
> -- [https://biocontainers-edu.readthedocs.io/en/latest/what_is_biocontainers.html](https://biocontainers-edu.readthedocs.io/en/latest/what_is_biocontainers.html)
{: .quote}

Singularity is an alternative to Docker that is much friendlier for HPCs

> Singularity is a container platform. It allows you to create and run containers that package up pieces of software in a way that is portable and reproducible.
>
> -- [https://sylabs.io/guides/3.7/user-guide/introduction.html](https://sylabs.io/guides/3.7/user-guide/introduction.html)
{: .quote}

> ### Agenda
>
> 1. TOC
> {:toc}
>
{: .agenda}

# Installing Singularity

First, we will install Singularity using Ansible. On most operating systems there is no package for singularity yet, so we must use a role which will compile it from source. If you're on CentOS7/8, it is available through the EPEL repository.

> ### {% icon tip %} CentOS7
> If you are using CentOS7, you can skip this hands-on section and instead install the `epel-release` and `singularity` system packages in your `pre_tasks`.
{: .tip}

> ### {% icon hands_on %} Hands-on: Installing Singularity with Ansible
>
> 1. In your working directory, add the Singularity role to your `requirements.yml` file:
>
>    ```yaml
>    - src: cyverse-ansible.singularity
>      version: 048c4f178077d05c1e67ae8d9893809aac9ab3b7
>    - src: gantsign.golang
>      version: 2.6.3
>    ```
>
> 2. Install the requirements with `ansible-galaxy`:
>
>    ```console
>    ansible-galaxy role install -p roles -r requirements.yml
>    ```
>
> 4. Specify which version of Singularity you want to install, in `group_vars/galaxyservers.yml`:
>
>    {% raw %}
>    ```yaml
>    # Golang
>    golang_gopath: '/opt/workspace-go'
>    # Singularity target version
>    singularity_version: "3.7.0"
>    singularity_go_path: "{{ golang_install_dir }}"
>    ```
>    {% endraw %}
> 4. Add the new roles to your `galaxy.yml` playbook, before the Galaxy server itself. We'll do this bceause it's a dependency of Galaxy to run, so it needs to be there before Galaxy starts.
>
>    ```diff
>    --- a/galaxy.yml
>    +++ b/galaxy.yml
>    @@ -16,6 +16,8 @@
>           become: true
>           become_user: postgres
>         - geerlingguy.pip
>    +    - gantsign.golang
>    +    - cyverse-ansible.singularity
>         - galaxyproject.galaxy
>         - role: uchida.miniconda
>           become: true
>    ```
>
> 5. Run the playbook
>
>    > ### {% icon code-in %} Input: Bash
>    > ```
>    > ansible-playbook galaxy.yml
>    > ```
>    {: .code-in}
>
> 6. Singularity should now be installed on your Galaxy server. You can test this by connecting
> to your server and run the following command:
>
>    > ### {% icon code-in %} Input: Bash
>    > ```
>    > singularity run docker://hello-world
>    > ```
>    {: .code-in}
>
>    > ### {% icon code-out %} Output: Bash
>    > ```
>    > INFO:    Converting OCI blobs to SIF format
>    > INFO:    Starting build...
>    > Getting image source signatures
>    > Copying blob 0e03bdcc26d7 done
>    > Copying config b23a8f6569 done
>    > Writing manifest to image destination
>    > Storing signatures
>    > 2021/01/08 11:25:12  info unpack layer: sha256:0e03bdcc26d7a9a57ef3b6f1bf1a210cff6239bff7c8cac72435984032851689
>    > INFO:    Creating SIF file...
>    > WARNING: passwd file doesn't exist in container, not updating
>    > WARNING: group file doesn't exist in container, not updating
>    >
>    > Hello from Docker!
>    > This message shows that your installation appears to be working correctly.
>    > ...
>    > ```
>    {: .code-out}
{: .hands_on}

## Configure Galaxy to use Singularity

Now, we will configure Galaxy to run tools using Singularity containers, which will be automatically fetched from [the BioContainers repository](https://quay.io/organization/biocontainers).

> ### {% icon hands_on %} Hands-on: Configure Galaxy to use Singularity
>
> 1. Edit the `group_vars/galaxyservers.yml` file and add a `dependency_resolvers_config_file` entry and a corresponding `galaxy_config_files` entry:
>
>    {% raw %}
>    ```diff
>    --- a/group_vars/galaxyservers.yml
>    +++ b/group_vars/galaxyservers.yml
>    @@ -28,6 +28,7 @@ miniconda_manage_dependencies: false
>
>     galaxy_config:
>       galaxy:
>    +    dependency_resolvers_config_file: "{{ galaxy_config_dir }}/dependency_resolvers_conf.xml"
>    +    containers_resolvers_config_file: "{{ galaxy_config_dir }}/container_resolvers_conf.xml"
>         brand: "🧬🔬🚀"
>         admin_users: admin@example.org
>         database_connection: "postgresql:///galaxy?host=/var/run/postgresql"
>    @@ -65,6 +66,11 @@ galaxy_config_templates:
>       - src: templates/galaxy/config/job_conf.xml.j2
>         dest: "{{ galaxy_config.galaxy.job_config_file }}"
>
>    +galaxy_config_files:
>    +- src: files/galaxy/config/dependency_resolvers_conf.xml
>    +  dest: "{{ galaxy_config.galaxy.dependency_resolvers_config_file }}"
>    +- src: files/galaxy/config/container_resolvers_conf.xml
>    +  dest: "{{ galaxy_config.galaxy.containers_resolvers_config_file }}"
>    ```
>    {% endraw %}
>
> 2. Create the `files/galaxy/config` directory if it doesn't exist:
>
>    > ### {% icon code-in %} Input: Bash
>    > ```
>    > mkdir -p files/galaxy/config
>    > ```
>    {: .code-in}
>
> 3. Create the new file `files/galaxy/config/dependency_resolvers_conf.xml`. This will not enable any dependency resolvers like the legacy toolshed packages or Galaxy packages, and instead everything will be resolved through Singularity.
>
>    ```xml
>    <dependency_resolvers>
>    </dependency_resolvers>
>    ```
>
> 3. Create the new file `files/galaxy/config/container_resolvers_conf.xml`, this specifies the order in which to attempt container resolution.
>
>    ```xml
>    <containers_resolvers>
>      <explicit_singularity />
>      <cached_mulled_singularity />
>      <mulled_singularity auto_install="False"/>
>      <build_mulled_singularity auto_install="False"/>
>    </containers_resolvers>
>    ```
> 3. Now, we want to make Galaxy run jobs using Singularity. Modify the file `templates/galaxy/config/job_conf.xml.j2`, by adding the `singularity_enabled` parameter:
>
>    ```diff
>    --- a/templates/galaxy/config/job_conf.xml.j2
>    +++ b/templates/galaxy/config/job_conf.xml.j2
>    @@ -3,7 +3,9 @@
>         </plugins>
>    -    <destinations default="local_destination">
>    +    <destinations default="singularity">
>             <destination id="local_destination" runner="local_plugin"/>
>    +        <destination id="singularity" runner="local_plugin">
>    +            <param id="singularity_enabled">true</param>
>    +            <!-- Ensuring a consistent collation environment is good for reproducibility. -->
>    +            <env id="LC_ALL">C</env>
>    +            <!-- The cache directory holds the docker containers that get converted. -->
>    +            <env id="SINGULARITY_CACHEDIR">/tmp/singularity</env>
>    +            <!-- Singularity uses a temporary directory to build the squashfs filesystem. -->
>    +            <env id="SINGULARITY_TMPDIR">/tmp</env>
>    +        </destination>
>         </destinations>
>         <tools>
>         </tools>
>    ```
>
> 4. Re-run the playbook
>
>    > ### {% icon code-in %} Input: Bash
>    > ```
>    > ansible-playbook galaxy.yml
>    > ```
>    {: .code-in}
>
> 5. In your Galaxy admin interface, install the minimap2 tool.
>
>    - Login to Galaxy as the admin user
>    - Click the "admin" menu at the top
>    - Under "Tool Management" on the left select "Install and Uninstall"
>    - search for `minimap2` and install the latest version with the Target Section "Mapping"
>
>    ![Screenshot of the install interface, minimap2 is entered in the search box and the latest revision shows it is currently cloning](../../images/install-minimap2.png)
>
> 6. Upload the following fasta file
>
>    ```
>    >testing
>    GATTACAGATHISISJUSTATESTGATTACA
>    ```
>
> 2. **Map with minimap2** {% icon tool %} with the following parameters
>    - *"Will you select a reference genome from your history or use a built-in index"*: `Use a genome from history and build index`
>    - *"Use the following dataset as the reference sequence"*: The fasta file you uploaded
>    - *"Single or Paired-end reads"*: `Single`
>        - {% icon param-file %} *"Select fastq dataset"*: The fasta file you uploaded
>
>    Your job should be executed using Singularity with a BioContainer! You can watch the logs of Galaxy to see this happening.
>
>    > ### {% icon code-in %} Input: Bash
>    > ```
>    > journalctl -f
>    > ```
>    {: .code-in}
>
>    > ### {% icon code-out %} Output
>    > ```
>    > uwsgi[1190010]: galaxy.tool_util.deps.containers INFO 2021-01-08 13:37:30,342 [p:1190010,w:0,m:2] [LocalRunner.work_thread-1] Checking with container resolver [MulledSingularityContainerResolver[namespace=biocontainers]] found description [ContainerDescription[identifier=docker://quay.io/biocontainers/mulled-v2-66534bcbb7031a148b13e2ad42583020b9cd25c4:e1ea28074233d7265a5dc2111d6e55130dff5653-0,type=singularity]]
>    > uwsgi[1190010]: galaxy.jobs.command_factory INFO 2021-01-08 13:37:30,418 [p:1190010,w:0,m:2] [LocalRunner.work_thread-1] Built script [/srv/galaxy/jobs/000/23/tool_script.sh] for tool command [minimap2 --version > /srv/galaxy/jobs/000/23/outputs/COMMAND_VERSION 2>&1; ln -f -s '/data/000/dataset_22.dat' reference.fa && minimap2           -t ${GALAXY_SLOTS:-4} reference.fa '/data/000/dataset_22.dat' -a | samtools sort -@${GALAXY_SLOTS:-2} -T "${TMPDIR:-.}" -O BAM -o '/data/000/dataset_23.dat' > '/data/000/dataset_23.dat']
>    > uwsgi[1190010]: galaxy.jobs.runners DEBUG 2021-01-08 13:37:30,441 [p:1190010,w:0,m:2] [LocalRunner.work_thread-1] (23) command is: mkdir -p working outputs configs
>    > uwsgi[1190010]: if [ -d _working ]; then
>    > uwsgi[1190010]:     rm -rf working/ outputs/ configs/; cp -R _working working; cp -R _outputs outputs; cp -R _configs configs
>    > uwsgi[1190010]: else
>    > uwsgi[1190010]:     cp -R working _working; cp -R outputs _outputs; cp -R configs _configs
>    > uwsgi[1190010]: fi
>    > uwsgi[1190010]: cd working; SINGULARITYENV_GALAXY_SLOTS=$GALAXY_SLOTS SINGULARITYENV_HOME=$HOME SINGULARITYENV__GALAXY_JOB_HOME_DIR=$_GALAXY_JOB_HOME_DIR SINGULARITYENV__GALAXY_JOB_TMP_DIR=$_GALAXY_JOB_TMP_DIR SINGULARITYENV_TMPDIR=$TMPDIR SINGULARITYENV_TMP=$TMP SINGULARITYENV_TEMP=$TEMP singularity -s exec -B /srv/galaxy/server:/srv/galaxy/server:ro -B /srv/galaxy/var/shed_tools/toolshed.g2.bx.psu.edu/repos/iuc/minimap2/8c6cd2650d1f/minimap2:/srv/galaxy/var/shed_tools/toolshed.g2.bx.psu.edu/repos/iuc/minimap2/8c6cd2650d1f/minimap2:ro -B /srv/galaxy/jobs/000/23:/srv/galaxy/jobs/000/23 -B /srv/galaxy/jobs/000/23/outputs:/srv/galaxy/jobs/000/23/outputs -B /srv/galaxy/jobs/000/23/configs:/srv/galaxy/jobs/000/23/configs -B /srv/galaxy/jobs/000/23/working:/srv/galaxy/jobs/000/23/working -B /data:/data -B /srv/galaxy/var/tool-data:/srv/galaxy/var/tool-data:ro -B /srv/galaxy/var/tool-data:/srv/galaxy/var/tool-data:ro --home $HOME:$HOME docker://quay.io/biocontainers/mulled-v2-66534bcbb7031a148b13e2ad42583020b9cd25c4:e1ea28074233d7265a5dc2111d6e55130dff5653-0 /bin/bash /srv/galaxy/jobs/000/23/tool_script.sh > ../outputs/tool_stdout 2> ../outputs/tool_stderr; return_code=$?; cd '/srv/galaxy/jobs/000/23';
>    > ```
>    {: .code-out.code-max-300}
>
{: .hands_on}

<!--
## Use Singularity containers from CVMFS

Galaxy can be configured to use pre-made Singularity containers available from /cvmfs/singularity.galaxyproject.org/.
In order to do so, you will first need to set up CVMFS by doing the [CVMFS]({{ site.baseurl }}/topics/admin/tutorials/cvmfs/tutorial.html) tutorial.
After finishing the CVMFS tutorial, come back, and do this hands-on.

> ### {% icon hands_on %} Optional: Hands-on: Configure Galaxy to use Singularity containers from CVMFS
>
> 1. Edit the `group_vars/galaxyservers.yml` file and add `containers_resolvers_config_file` and `galaxy_singularity_images_cvmfs_path`:
>{% raw %}
>    ```yaml
>    galaxy_singularity_images_cvmfs_path: "/cvmfs/singularity.galaxyproject.org/all/"
>    galaxy_config:
>      galaxy:
>        ...
>        containers_resolvers_config_file: "{{ galaxy_config_dir }}/container_resolvers_conf.xml"
>    ```
>{% endraw %}
>
> 2. Also in `group_vars/galaxyservers.yml`, add a `galaxy_config_templates` entry:
>{% raw %}
>    ```yaml
>    galaxy_config_templates:
>      - src: templates/galaxy/config/container_resolvers_conf.xml.j2
>        dest: "{{ galaxy_config_dir }}/container_resolvers_conf.xml"
>    ```
>{% endraw %}
>
> 3. Create the new file `templates/galaxy/config/container_resolvers_conf.xml.j2`:
>{% raw %}
>    ```xml
>    <containers_resolvers>
>        <explicit_singularity />
>        <cached_mulled_singularity cache_directory="{{ galaxy_singularity_images_cvmfs_path }}"/>
>    </containers_resolvers>
>    ```
>{% endraw %}
>
> 4. Re-run the playbook (`ansible-playbook galaxy.yml`)
>
> 5. Rerun your minimap2 job from the previous section. This time it should use a container served by CVMFS.
>
{: .hands_on}

-->
