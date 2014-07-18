#HowTo for multiple database connection in rails

## Using multiple connections within a single model (method)

Your config/database.yml file

    default: &default
      adapter: mysql2 # or postgresql
      encoding: utf8
      database: your_db
      pool: 5
      username: your_db_username
      password: your_db_password
      host: 192.168.0.1
    
    production:
      <<: *default
      
    production_fork:
      <<: *default
      database: your_forked_db
      host: 192.168.0.2
      
    production_n:
      ...
    
Your codeode which will work with multiple connections

    # Use pipe to transfer data between fork and parent process
    reader, writer = IO.pipe

    # Fork process to isolate the work with multiple database connections from application
    pid = fork do
      reader.close()
      
      begin
        # if used Redis uncomment this line to reset connection
        # REDIS = Redis.new(:host => '', :port => '', :password => '')
        
        # In this point used application current database connections, :production
        # Fetch data from your_db database
        @model = YourModels.includes(:another_model).first

        # Switch database connection, in this point start using :production_fork connection
        ActiveRecord::Base.establish_connection :production_fork 
        @cloned_model = @model.dup
        
        # Model has been saved with :production_fork connection
        # Data has been saved in your_fork_db database
        @cloned_model.save
        
        writer.write(@cloned_model.id)
      rescue Exception => e
        print e.inspect
      ensure
        ActiveRecord::Base.remove_connection
        
        # If used Redis uncomment this line to disconnect
        # REDIS.client.disconnect if used
        
        Process.exit!
      end
    end
    
    Process.waitpid(pid, 0)
    
    # In this point used again application current database connection, :production
    
    writer.close()
    
    # Print @cloned_model.id
    print reader.read()






