$node_name = "app-server-#{Time.now.strftime("%y-%m-%d-%H-%M")}"

set_aws_env_vars

TestLoadBalancer = 'production-test'
ProductionLoadBalancers = ['prod-elb-1', 'prod-elb-2']

NewAppServerTag = 'new-app-server'
CurrentAppServerTag = 'hot-app-server'

desc "creates 3 new app servers."
task  :create_new_app_servers do
  threads = []
  instance_ids = []
  3.times do |i|
    threads << Thread.new { instance_ids << create_app_server(i) }
    sleep 30 # a bit of breathing room before firing up a new app server
  end
  threads.each{ |th| th.join }
  register_instances_with_load_balancer(TestLoadBalancer, instance_ids)
  verify_instance_health(TestLoadBalancer)
end

desc "Rolls in new app servers to production servers."
task :deploy do
  log "Registering the new app servers with the production load balancers..."
  register_instances_with_load_balancer(ProductionLoadBalancers, get_instance_ids_from_tags(NewAppServerTag))

  log "Verifying instance health..."
  ProductionLoadBalancers.each {|lb| verify_instance_health(lb) }

  log "Removing old app servers from production..."
  deregister_instances_with_load_balancer(ProductionLoadBalancers, get_instance_ids_from_tags(CurrentAppServerTag))

  log "Removing new app servers from test load balancer..."
  deregister_instances_with_load_balancer(TestLoadBalancer, get_instance_ids_from_tags(NewAppServerTag))
end

desc "Rolls back and puts in previous app servers into production"
task :rollback do
  log "Re-registering the old app servers with the production load balancer..."
  register_instances_with_load_balancer(ProductionLoadBalancers, get_instance_ids_from_tags(CurrentAppServerTag))

  log "Verifying old app server health..."
  ProductionLoadBalancers.each {|lb| verify_instance_health(lb) }

  log "Deregistering new app servers with production load balancers..."
  deregister_instances_with_load_balancer(ProductionLoadBalancers, get_instance_ids_from_tags(NewAppServerTag))

  log "Registering new instances with the test load balancer..."
  register_instances_with_load_balancer(TestLoadBalancer, get_instance_ids_from_tags(NewAppServerTag))
end

desc "Commits the app server tags, deploy looks good."
task :cleanup do
  log "Terminating hold hot-app-server instances..."
  terminate_instances(get_instance_ids_from_tags(CurrentAppServerTag))

  log "Tagging new instances as the current hot app servers..."
  tag_instances(CurrentAppServerTag, get_instance_ids_from_tags(NewAppServerTag))
end

#######################################
# Low-level methods
#######################################

def log(msg)
  $stdout << msg + "\n"
end

def get_node_name(server_num)
  "#{$node_name}-#{server_num}"
end

def get_instance_ids_from_tags(node_name)
  `./ec2-api-tools/bin/ec2-describe-tags | grep "#{node_name}" | awk '{print $3}'`.split("\n")
end

def run(cmd)
  `#{cmd}`
  if $?.exitstatus != 0
    raise "Running #{cmd} failed with exit status #{$?.exitstatus}!\n"
  end
end

def create_app_server(server_num)
  log "Creating app server #{server_num}"
  log_path="/tmp/knife-ec2-#{Time.now.to_f}"
  run %Q(knife ec2 server create -VV -r "role[hybrid-app-server]" --flavor c1.xlarge --image ami-7aea5015 --groups pd-app-server --availability-zone us-east-1d --node-name "#{get_node_name(server_num)}" --ssh-key pd-app-server --ssh-user ec2-user -i ~/.ssh/pd-app-server --distro minimal 2>&1 >> #{log_path})

  instance_id = get_instance_ids_from_tags(get_node_name(server_num)).first
  log "Tagging instance #{instance_id}..."
  run %Q(./ec2-api-tools/bin/ec2-create-tags #{instance_id} --tag 'role=#{NewAppServerTag}')
  instance_id
end

def verify_instance_health(lb, retries = 4)
  output = `./elb-tools/bin/elb-describe-instance-health #{lb}`
  if output == "No instances currently registered to LoadBalancer\n"
    $stdout << output
    return
  end

  retries.times do
    sleep 5
    output = `./elb-tools/bin/elb-describe-instance-health #{lb}| awk '{print $3}'`.map {|o| o.strip }
    if output.all? {|o| o == "InService" }
      log "All servers are healthy."
      return
    else
      unhealthy_count = output.select {|o| o != "InService" }.size
      log "#{unhealthy_count}/#{output.count} instances are reporting not healthy. waiting to retry..."
    end
  end
  raise "Not all instances healthy!"
end

def register_instances_with_load_balancer(lbs, instance_ids)
  Array(lbs).each do |lb|
    run %Q(./elb-tools/bin/elb-register-instances-with-lb #{lb} --instances #{instance_ids.join(' ')})
  end
end

def deregister_instances_with_load_balancer(lbs, instance_ids)
  Array(lbs).each do |lb|
    run %Q(./elb-tools/bin/elb-deregister-instances-from-lb #{lb} --instances #{instance_ids.join(' ')})
  end
end

def tag_instances(tag, *instance_ids)
  instance_ids.flatten.each do |instance|
    run %Q(./ec2-api-tools/bin/ec2-create-tags #{instance} --tag 'role=#{tag}')
  end
end

def terminate_instances(instance_ids)
  run %Q(./ec2-api-tools/bin/ec2-terminate-instances #{instance_ids.join(' ')})
end

def set_aws_env_vars
  ENV['AWS_ELB_HOME'] ||= "#{File.expand_path(File.dirname(__FILE__))}/elb-tools"
  ENV['AWS_CREDENTIAL_FILE'] ||= "#{File.expand_path(File.dirname(__FILE__))}/aws-credentials"
  File.read(ENV['AWS_CREDENTIAL_FILE']).each do |line|
    ENV['AWS_ACCESS_KEY'] ||= $1 if line =~ /^\s*AWSAccessKeyId\s*=\s*(\S+)/
    ENV['AWS_SECRET_KEY'] ||= $1 if line =~ /^\s*AWSSecretKey\s*=\s*(\S+)/
  end
end
