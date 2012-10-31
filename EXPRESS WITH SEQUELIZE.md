EXPRESS WITH SEQUELIZE
======================

Well, as [this](http://stackoverflow.com/questions/12487416/example-express-app-that-uses-sequelize)
guy said is impossible to put your models in different files due to circular references 
and multiple sequelize instantiation(?). So, creating a single file is mandatory in order to create 
complex relationships. 

**But**, you can strive this by putting your models data in different files but setting up 
sequelize in a single module which loads all the model definitions and holds all the model classes. 
Obviusly you must have this module as a [singleton](http://simplapi.wordpress.com/2012/05/14/node-js-singleton-structure/).

The idea is to have something like this for your model files:

Hello.js:

    //Getting the orm instance
    var orm = require("path/to/lib/model")
    , Seq = orm.Seq();
    
    //Creating our module
    module.exports = {
        model:{
            id: Seq.INTEGER,
            name: Seq.STRING
        },
        relations:{
           hasMany:"World" 
        },
        options:{
            freezeTableName: true
        }
    }

Note that there is no sequelize model initialization, you just give the data necesary to initialize 
the model. That `orm` object is indeed our singleton and, in this case, just returns the `Sequelize` 
constructor which is needed for several things like setting the model's field types.

At our app.js file we just create and setup our orm singleton:

app.js:

    ...
	//Setting up our singleton...
    require("path/to/lib/model").setup('./app/models', "db_name", "user", "pass", {
        host: 'my.host'
		....
    });
    ...

Then you can use your model in your controller files:

Index.js:

    //Getting our singleton
    var orm = require("path/to/lib/model");

    exports.hello = function (request, response){
        //Getting the Hello model from the orm.
        var Hello = orm.model("Hello");
        //Doing complex stuff like getting entries from a many-to-many relationship...
        Hello.find(1).success(function (hello){
            hello.getWorlds().success(function (worlds) {
                response.send(worlds)   
            });
        });
    }
	
And finally our nitty singleton class which holds all the models:

path/to/lib/model.js:

	var filesystem = require('fs');
	var models = {};
	var relationships = {};

	var singleton = function singleton(){
		var Sequelize = require("sequelize");
		var sequelize = null;
		var modelsPath = "";
		this.setup = function (path, database, username, password, obj){
			modelsPath = path;
			
			if(arguments.length == 3){
				sequelize = new Sequelize(database, username);
			}
			else if(arguments.length == 4){
				sequelize = new Sequelize(database, username, password);
			}
			else if(arguments.length == 5){
				sequelize = new Sequelize(database, username, password, obj);
			}        
			init();
		}
		
		this.model = function (name){
			return models[name];
		}
		
		this.Seq = function (){
			return Sequelize;
		}
		
		function init() {
			filesystem.readdirSync(modelsPath).forEach(function(name){
				var object = require(modelsPath + "/" + name);
				var options = object.options || {}
				var modelName = name.replace(/\.js$/i, "");
				models[modelName] = sequelize.define(modelName, object.model, options);
				if("relations" in object){
					relationships[modelName] = object.relations;
				}
			});
			console.log(models)
			for(var name in relationships){
				var relation = relationships[name];
				for(var relName in relation){
					var related = relation[relName];
					console.log(related)
					models[name][relName](models[related]);
				}
			}
		}
			
		if(singleton.caller != singleton.getInstance){
			throw new Error("This object cannot be instanciated");
		}
	}

	singleton.instance = null;

	singleton.getInstance = function(){
		if(this.instance === null){
			this.instance = new singleton();
		}
		return this.instance;
	}

	module.exports = singleton.getInstance();
	
This Module has a very simple interface, it has just three methods:

	setup(path, database, username, password, obj)

Instantiates the `sequelize` object, it has the same parameters as the `Sequelize` constructors plus
a `path` that indicates the models folder.

	model(name)
	
Gets the Model class with the given name. The name is the same of the file that contains the model without extension.

	Seq()
	
Returns the `Sequelize` constructor.

Well, that's all for now. I hope this article have saved some lives :)