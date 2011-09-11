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
set :ami, `curl http://mng.iop.kcl.ac.uk/cass_data/buckley_ami/AMIID`.chomp
#set :ami, 'ami-52794c26' #32-bit ubuntu lucid server (eu-west-1)
set :instance_type, 'm1.large'
set :working_dir, '/mnt/work'

set :group_name, 'REST_ChIP_mouse_astrocytes_volta'
set :nhosts, 1

set :snap_id, `cat SNAPID`.chomp #empty until you've created a snapshot
set :vol_id, `cat VOLUMEID`.chomp #empty until you've created a new volume
set :ebs_size, 12  
set :availability_zone, 'eu-west-1a'  #wherever your ami is. 
#set :dev, '/dev/sdf'
set :dev, '/dev/xvdf'
set :mount_point, '/mnt/data'
set :script_dir, '/mnt/work/scripts'
set :git_url, 'https://github.com/cassj/REST_ChIP_mouse_astrocytes_volta/raw/master'
# Try and load a local config file to override any of the above values, should one exist.
# So that if you change these values, they don't get overwritten if you update the repos.
begin
 load("Capfile.local")
rescue Exception
end



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

## really these should have been derived from the raw data but
## I'd already run them and I don't have time to redo everything 
#
#desc "upload BAM files"
#task :upload_bam, :roles => group_name do
#  run "mkdir #{mount_point}/BAM"
#  `cd ToBAM/results && tar -cvzf bamfiles.tgz *.ba*`
#  upload("ToBAM/results/bamfiles.tgz", "#{mount_point}/BAM/bamfiles.tgz")
#  run "cd #{mount_point}/BAM && tar -xvzf bamfiles.tgz"
#  run "cd #{mount_point}/BAM && rm bamfiles.tgz"
#end
#before 'upload_bam','EC2:start'
#
#
#desc "upload Macs files"
#task :upload_macs, :roles => group_name do
#  run "mkdir #{mount_point}/Macs"
#  `cd Macs/results && tar -cvzf macsfiles.tgz *`
#  upload("Macs/results/macsfiles.tgz", "#{mount_point}/Macs/macsfiles.tgz")
#  run "cd #{mount_point}/Macs && tar -xvzf macsfiles.tgz"
#  run "cd #{mount_point}/Macs && rm macsfiles.tgz"
#end
#before 'upload_macs','EC2:start'



# Have to start from scratch as the Input BAM file is damaged

desc "Upload data files"
task :upload_data, :roles => group_name do
    puts "rsync -e 'ssh -i /home/cassj/ec2/cassj.pem' -vzP data/* ubuntu@ec2-46-51-135-144.eu-west-1.compute.amazonaws.com:#{mount_point}" 
end
before 'upload_data', 'EC2:start'


desc "remove .fa from chr names"
task :remove_fa, :roles => group_name do
  run "perl -pi.bak -e 's/\\.fa//g;' #{mount_point}/*.txt"
end
before "remove_fa", "EC2:start"




desc "install liftOver"
task :install_liftover, :roles => group_name do

  #liftover
  sudo 'wget -O /usr/bin/liftOver http://hgdownload.cse.ucsc.edu/admin/exe/linux.x86_64/liftOver'
  sudo 'chmod +x /usr/bin/liftOver'

  #chainfile - note that the script sortedmm8tomm9 expects the chain files to be in ../lib
  run "mkdir -p #{working_dir}/lib"
  run "wget  -O #{working_dir}/lib/mm8ToMm9.over.chain.gz 'http://hgdownload.cse.ucsc.edu/goldenPath/mm8/liftOver/mm8ToMm9.over.chain.gz'"
  run "gunzip -c #{working_dir}/lib/mm8ToMm9.over.chain.gz > #{working_dir}/lib/mm8ToMm9.over.chain"

  run "mkdir -p #{script_dir}"
  run "wget -O #{script_dir}/sortedmm8tomm9.R 'https://github.com/cassj/REST_ChIP_mouse_astrocytes_volta/raw/master/scripts/sortedmm8tomm9.R'"
#  run "wget -O #{script_dir}/sortedmm8tomm9.R  '#{git_url}/sortedmm8tomm9.R'"
  run "chmod +x #{script_dir}/sortedmm8tomm9.R"

end 
before "install_liftover", "EC2:start"


desc "liftOver ELAND mm8 positions to mm9"
task :liftOver, :roles => group_name do

  run "cd #{mount_point} && Rscript #{script_dir}/sortedmm8tomm9.R CMN025_180_unique_hits.txt"
  run "cd #{mount_point} && Rscript #{script_dir}/sortedmm8tomm9.R CMN026_026_unique_hits.txt"

end
before "liftOver", "EC2:start"


desc "Remove anything mapping to a random chr"
task :remove_random, :roles => group_name do
  run "cd #{mount_point} && perl -ni.bak -e 'print $_ unless /.*random.*/;' *_mm9.txt"
end
before "remove_random", "EC2:start"



desc "Convert sorted to SAM format"
task :to_sam, :roles => group_name do
  run "wget  -O #{script_dir}/sorted2sam.pl 'https://github.com/cassj/REST_ChIP_mouse_astrocytes_volta/raw/master/scripts/sorted2sam.pl'"   
  run "chmod +x #{script_dir}/sorted2sam.pl"
  run "cd #{mount_point} && perl #{script_dir}/sorted2sam.pl /mnt/data/CMN025_180_unique_hits_mm9.txt > /mnt/data/CMN025_180_unique_hits_mm9.sam"
  run "cd #{mount_point} && perl #{script_dir}/sorted2sam.pl /mnt/data/CMN026_026_unique_hits_mm9.txt > /mnt/data/CMN026_026_unique_hits_mm9.sam"

end
before "to_sam", "EC2:start"

desc "Convert SAM to BAM"
task :to_bam, :roles => group_name do
   run "wget -O #{script_dir}/mm9_lengths  '#{git_url}/scripts/mm9_lengths'"
   run "samtools view -bt #{script_dir}/mm9_lengths -o /mnt/data/CMN025_180_unique_hits_mm9.bam /mnt/data/CMN025_180_unique_hits_mm9.sam"
   run "samtools view -bt #{script_dir}/mm9_lengths -o /mnt/data/CMN026_026_unique_hits_mm9.bam /mnt/data/CMN026_026_unique_hits_mm9.sam"
end
before "to_bam", "EC2:start"


desc "Sort BAM"
task :sort_bam, :roles => group_name do
   run "samtools sort /mnt/data/CMN025_180_unique_hits_mm9.bam /mnt/data/CMN025_180_unique_hits_mm9_sorted"
   run "samtools sort /mnt/data/CMN026_026_unique_hits_mm9.bam /mnt/data/CMN026_026_unique_hits_mm9_sorted"
end
before "sort_bam", "EC2:start"

desc "Remove PCR Duplicates"
task :rm_dups, :roles => group_name do
  run "samtools rmdup -s /mnt/data/CMN025_180_unique_hits_mm9_sorted.bam /mnt/data/CMN025_180_unique_hits_mm9_sorted_nodup.bam"
  run "samtools rmdup -s /mnt/data/CMN026_026_unique_hits_mm9_sorted.bam /mnt/data/CMN026_026_unique_hits_mm9_sorted_nodup.bam"
end
before "rm_dups", "EC2:start"

desc "Index BAM files"
task :index_bam, :roles => group_name do
  run "samtools index /mnt/data/CMN025_180_unique_hits_mm9_sorted_nodup.bam"
  run "samtools index /mnt/data/CMN026_026_unique_hits_mm9_sorted_nodup.bam"
end
before "index_bam", "EC2:start"

desc "get results"
task :get_results, :roles => group_name do  
  system("mkdir -p results")
  download("/mnt/data/CMN025_180_unique_hits_mm9_sorted_nodup.bam", "results/CMN025_180_unique_hits_mm9_sorted_nodup.bam")
  download("/mnt/data/CMN025_180_unique_hits_mm9_sorted_nodup.bam.bai", "results/CMN025_180_unique_hits_mm9_sorted_nodup.bam.bai")
  download("/mnt/data/CMN026_026_unique_hits_mm9_sorted_nodup.bam", "results/CMN026_026_unique_hits_mm9_sorted_nodup.bam")
  download("/mnt/data/CMN026_026_unique_hits_mm9_sorted_nodup.bam.bai", "results/CMN026_026_unique_hits_mm9_sorted_nodup.bam.bai")
end 
before "get_results","EC2:start"

#### Macs

######## Peak Finding

#Need to be able to specify multiple treatment and control pairs here.

task :run_macs, :roles => group_name do
  
  treatment = "#{mount_point}/CMN025_180_unique_hits_mm9_sorted_nodup.bam"
  control = "#{mount_point}/CMN026_026_unique_hits_mm9_sorted_nodup.bam"
  genome = 'mm'
  bws = [300] 
  pvalues = [0.00001]

  #unsure what p values and bandwidths are appropriate, try a few?
  bws.each {|bw|
    pvalues.each { |pvalue|

      dir = "#{mount_point}/macs_#{bw}_#{pvalue}"
      run "rm -Rf #{dir}"
      run "mkdir #{dir}"
  
      macs_cmd =  "macs --treatment #{treatment} --control #{control} --name #{group_name} --format BAM --gsize #{genome} --bw #{bw} --pvalue #{pvalue}"
      run "cd #{dir} && #{macs_cmd}"
      
      dir = "#{mount_point}/macs_#{bw}_#{pvalue}_subpeaks"
      run "rm -Rf #{dir}"
      run "mkdir #{dir}"

      # With SubPeak finding
      # this will take a lot longer as you have to save the wig file
      macs_cmd =  "macs --treatment #{treatment} --control #{control} --name #{group_name} --format BAM --gsize #{genome} --call-subpeaks  --wig --bw #{bw} --pvalue #{pvalue}"
      run "cd #{dir} && #{macs_cmd}"
  
    }
  }

end
before 'run_macs', 'EC2:start'



desc "convert peaks to IRanges"
task :peaks_to_iranges, :roles => group_name do
  run "cd #{working_dir} && rm -f peaksBed2IRanges.R"
  upload('scripts/peaksBed2IRanges.R' , "#{working_dir}/peaksBed2IRanges.R")
  run "cd #{working_dir} && chmod +x peaksBed2IRanges.R"

  macs_dirs = capture "ls #{mount_point}"
  macs_dirs = macs_dirs.split("\n")
  macs_dirs = macs_dirs.select { |d| d =~ /.*macs.*/ }
  macs_dirs.each{|d|
    d.chomp
    unless d.match('tgz')
      xlsfiles = capture "ls #{mount_point}/#{d}/*peaks.xls"
      xlsfiles = xlsfiles.split("\n")
      xlsfiles.each{|x|
         run "cd #{mount_point}/#{d} && Rscript #{working_dir}/peaksBed2IRanges.R #{x}"
      }
    end
  }
end
before 'peaks_to_iranges', 'EC2:start'

desc "annotate IRanges"
task :annotate_peaks, :roles => group_name do
  run "cd #{working_dir} && rm -f mm9RDtoGenes.R"
  upload('scripts/mm9RDtoGenes.R', "#{working_dir}/mm9RDtoGenes.R")
  run "cd #{working_dir} && chmod +x mm9RDtoGenes.R"
  macs_dirs = capture "ls #{mount_point}"
  macs_dirs = macs_dirs.split("\n").select { |d| d =~ /.*macs.*/ }
  macs_dirs.each{|d|
    unless d.match('tgz')
      rdfiles = capture "ls #{mount_point}/#{d}/*RangedData.RData"
      rdfiles = rdfiles.split("\n")
      rdfiles.each{|rd|
        unless rd.match(/negative/)
          run "cd #{working_dir} && Rscript #{working_dir}/mm9RDtoGenes.R #{rd}"
        end
      }
    end
  }

end
before 'annotate_peaks', 'EC2:start'

task :pack_peaks, :roles => group_name do
  macs_dirs = capture "ls #{mount_point}"
  macs_dirs = macs_dirs.split("\n").select {|f| f.match(/.*macs.*/)}
  macs_dirs.each{|d|
    unless d.match('tgz')
      run "cd #{mount_point} &&  tar --exclude *_wiggle* -cvzf #{d}.tgz #{d}"
    end
  }

end
before 'pack_peaks','EC2:start'

task :get_peaks, :roles => group_name do
  macs_files = capture "ls #{mount_point}"
  macs_files = macs_files.split("\n").select {|f| f.match(/.*macs.*\.tgz/)}
  res_dir = 'results/alignment/bowtie/peakfinding/macs'
  `rm -Rf #{res_dir}`
  `mkdir -p #{res_dir}`
  macs_files.each{|f|
    download("#{mount_point}/#{f}", "#{res_dir}/#{f}")
    `cd #{res_dir} && tar -xvzf #{f}`
  }

end
before 'get_peaks', 'EC2:start'






######

# Validation Primers.
