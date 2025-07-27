# RZ-V2H-edge-impulse

I'll assume the required files (RTK0EF0180F05000SJ_linux-src.zip, nodejs_patches_for_EdgeImpulse_20240805.tar.gz, meta-ei.zip) are already downloaded there. The script is unchanged from the doc (no updates noted since Aug 2024), and it already enables dev features via EXTRA_IMAGE_FEATURES (including debug-tweaks, dev-pkgs, tools-debug, tools-sdk) plus SSH via debug-tweaks (which sets up SSH server with blank root password for dev ease—careful in prod, as it's insecure; you can set a root password later on the board with passwd)

# Group 1: Set up tracing, capture dir, create archive, move files.
set -x
DIR=`pwd`
mkdir ./archive
mv RTK* ./archive
mv nodejs_patches*gz ./archive
mv meta-ei.zip ./archive
