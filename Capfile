#### Congfiguration ####

# Note - this Capfile doesn't actually run the ToBAM and Macs stuff.
# I hadn't got the EBS stuff working when I ran them, so I just grabbed 
# the data back to our server. This file just uploads the data to an EBS
# vol so we can snapshot it for later analysis. If you need to rerun it 
# for some reason it probably makes sense to pull all the stuff in the 
# other Capfiles into this one. 

require 'catpaws/ec2'
require 'pathname'

set :aws_access_key,  ENV['AMAZON_ACCESS_KEY']
set :aws_secret_access_key , ENV['AMAZON_SECRET_ACCESS_KEY']
set :ec2_url, ENV['EC2_URL']
set :key, ENV['EC2_KEY'] #this should be the name of the key in aws, not the actual keyfile
set :ssh_options, {
  :keys => [ENV['EC2_KEYFILE']],
  :user => "ubuntu"
}
set :ami, 'ami-52794c26' #32-bit ubuntu lucid server (eu-west-1)
set :instance_type, 'm1.small'
set :working_dir, '/mnt/work'

set :group_name, 'REST_ChIP_mouse_astrocytes_volta'
set :nhosts, 1

set :snap_id, `cat SNAPID`.chomp #empty until you've created a snapshot
set :vol_id, `cat VOLUMEID`.chomp #empty until you've created a new volume
set :ebs_size, 12  
set :ebs_zone, 'eu-west-1a'  #wherever your ami is. 
set :dev, '/dev/sdf'
set :mount_point, '/mnt/data'

set :git_url, 'http://github.com/cassj/REST_ChIP_mouse_astrocytes_volta/raw/master'

#cap EC2:start
#cap EBS:create (unless you want to use the one that already exists)
#cap EBS:attach
#cap EBS:format_xfs (unless you're using a snapshot)
#cap EBS:mount_xfs
#
# run tasks as required
# 
#cap EBS:snapshot
#cap EBS:unmount
#cap EBS:detach
#cap EBS:delete
#cap EC2:stop


#### Tasks ####


desc "upload BAM files"
task :upload_bam, :roles => group_name do
  run "mkdir #{mount_point}/BAM"
  `cd ToBAM/results && tar -cvzf bamfiles.tgz *.ba*`
  upload("ToBAM/results/bamfiles.tgz", "#{mount_point}/BAM/bamfiles.tgz")
  run "cd #{mount_point}/BAM && tar -xvzf bamfiles.tgz"
  run "cd #{mount_point}/BAM && rm bamfiles.tgz"
end
before 'upload_bam','EC2:start'


desc "upload Macs files"
task :upload_macs, :roles => group_name do
  run "mkdir #{mount_point}/Macs"
  `cd Macs/results && tar -cvzf macsfiles.tgz *`
  upload("Macs/results/macsfiles.tgz", "#{mount_point}/Macs/macsfiles.tgz")
  run "cd #{mount_point}/Macs && tar -xvzf macsfiles.tgz"
  run "cd #{mount_point}/Macs && rm macsfiles.tgz"
end
before 'upload_macs','EC2:start'


