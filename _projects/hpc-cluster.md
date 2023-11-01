---
layout: project
title: 'Slurm HPC cluster'
caption: Development of a computation cluster from scratch.
description: >
  This project was developed to improve an existing computational cluster at Universitat de Barcelona.
date: '30-12-2022'
image: 
  path: /assets/img/projects/server2.jpg
  srcset: 
    1920w: /assets/img/projects/server2.jpg
    
links:
  - title: Link
    url: https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=&cad=rja&uact=8&ved=2ahUKEwiIg7ukjZ6CAxVPTKQEHeecCEoQFnoECBcQAQ&url=https%3A%2F%2Fupcommons.upc.edu%2Fbitstream%2Fhandle%2F2117%2F384973%2F173040.pdf%3Fsequence%3D2%26isAllowed%3Dy&usg=AOvVaw0x5URCKuPxfG_cWic96Kx6&opi=89978449
sitemap: false
---
I recently completed a project where I developed a Slurm CPU+GPU HPC cluster based on linux systems. The goal of the project was to create a high-performance 
computing environment that would allow users to run CPU and GPU-intensive tasks in a seamless and efficient manner.

To achieve this, I started by setting up a Kerberos authentication system to provide secure access to the cluster. I then configured the RAID array to ensure data redundancy and fast read/write speeds. 
Next, I installed Slurm to manage job scheduling and resource allocation, and integrated GPU support to allow for efficient parallel processing.

To make the environment as user-friendly as possible, I also installed EasyBuild to provide an automated package installation system, and Lmod to allow users to easily load and manage software modules. 
This allowed users to quickly and easily set up their own computing environments with all the necessary tools and libraries.

The end result was a powerful and user-friendly HPC cluster that was able to handle even the most demanding CPU and GPU workloads. With the addition of Kerberos, the system was also highly secure, 
ensuring that user data and resources were protected at all times. Overall, it was a challenging and rewarding project that allowed me to leverage my skills in system administration, high-performance computing, and user support.

If you are interested in the project details, you can download the PDF from the link above or find it on my GitHub.

Link to the repo: [https://github.com/marcfpalau/Slurm_cluster_deployment](https://github.com/marcfpalau/Slurm_cluster_deployment)
