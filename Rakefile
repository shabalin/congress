load 'analytics/report.rake'

task :environment do
  require 'rubygems'
  require 'bundler/setup'
  require 'config/environment'
  
  require 'tasks/report'
  require 'pony'
end

namespace :development do
  desc "Load a fake 'development' api key into the db"
  task :api_key => :environment do
    require 'analytics/api_key'
    
    if ApiKey.where(:key => "development", :email => "nobody@example.com").first.nil?
      ApiKey.create! :status => "A", :email => "nobody@example.com", :key => "development"
      puts "Created 'development' API key"
    else
      puts "'development' API key already exists"
    end
  end
end

desc "Run through each model and create all indexes" 
task :create_indexes => :environment do
  require 'analytics/api_key'
  
  begin
    models = Dir.glob('models/*.rb').map do |file|
      File.basename(file, File.extname(file)).camelize.constantize
    end + [ApiKey, Report]
    
    models.each do |model| 
      model.create_indexes
      puts "Created indexes for #{model}"
    end
  rescue Exception => ex
    email "Exception creating indexes, message and backtrace attached", {'message' => ex.message, 'type' => ex.class.to_s, 'backtrace' => ex.backtrace}.inspect
    puts "Error creating indexes, emailed report."
  end
end

desc "Set the crontab in place for this environment"
task :set_crontab => :environment do
  environment = ENV['environment']
  current_path = ENV['current_path']
  
  if environment.blank? or current_path.blank?
    puts "No environment or current path given, exiting."
    exit
  end
  
  if system("cat #{current_path}/config/cron/#{environment}.crontab | crontab")
    puts "Successfully overwrote crontab."
  else
    email "Crontab overwriting failed on deploy."
    puts "Unsuccessful in overwriting crontab, emailed report."
  end
end

# for each folder in tasks, generate a rake task
Dir.glob('tasks/*/').each do |file|
  name = File.basename file
  
  namespace :task do
    task name.to_sym => :environment do
      run_task name
    end
  end
end


def run_task(name)
  require 'tasks/utils'
  
  task_name = name.camelize
  
  start = Time.now
  
  begin
    if File.exist? "tasks/#{name}/#{name}.rb"
      run_ruby name
    elsif File.exist? "tasks/#{name}/#{name}.py"
      run_python name
    else
      raise Exception.new "Couldn't locate task file"
    end
    
  rescue Exception => ex
    Report.failure task_name, "Exception running #{name}, message and backtrace attached", {:elapsed_time => Time.now - start, :exception => {'message' => ex.message, 'type' => ex.class.to_s, 'backtrace' => ex.backtrace}}
    
  else
    complete = Report.complete task_name, "Completed running #{name}", {:elapsed_time => Time.now - start}
    puts complete
  end
  
  # go through any reports filed from the task, and email about any failures or warnings
  Report.unread.where(:source => task_name).all.each do |report|
    puts report
    email report if report.failure? or report.warning?
    report.mark_read!
  end

end

def run_ruby(name)
  load "tasks/#{name}/#{name}.rb"
  name.camelize.constantize.run :config => config, :args => ARGV[1..-1]
end

def run_python(name)
  system "python tasks/runner.py #{name} #{config[:mongoid]['host']} #{config[:mongoid]['database']} #{ARGV[1..-1].join ' '}"
end

def email(report, body = nil)
  if config[:email][:to] and config[:email][:to].any?
    begin
      if report.is_a?(Report)
        Pony.mail config[:email].merge(:subject => report.email_subject, :body => report.email_body)
      else
        Pony.mail config[:email].merge(:subject => report, :body => (body || report))
      end
    rescue Errno::ECONNREFUSED
      puts "Couldn't email report, connection refused! Check system settings."
    end
  end
end