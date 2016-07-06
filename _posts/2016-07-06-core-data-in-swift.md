---
layout: post
title:  "Use Core Data in UITableView in Swift 2.0"
date:   2016-07-06 10:45:00 +0800
categories: iOS
---
### Introduction
Core Data is the framework in iOS SDK that is used for permanent data storage for iOS apps. Essentially a SQLite database but embedded into the iOS developing environment in Xcode, it greatly simplifies the process of data management for personal-level apps.  

If Core Data is included in an iOS project, the entire database should be found by a `xcdatamodeld` file. It is consisted of multiple **entities** and **relationships** between them, in which are **attributes** for each entity. An entity is analogous to a class in memory-based programming, and each attribute in an entity is like a variable in that class. When storing data to one entity, the program is basically "instantiating" an object of this "class", and gives values to those "variables".  All objects of different entities could be stored in a `NSManagedObject` array, which is a dictionary-like data structure.  

### An Example
We are creating a database for all activities in a college campus, with every activity having a holder, a description and a date. First we create an `Activity` entity and it has attributes `holder`, `descript` and `date`. Suppose two activities are added to the database; the data structure will look like this:  

    An `NSManagedObject` array of entity `Activity`; item order does not matter:  
    |-------------------------------------------|   |--------------------------------------|
    | [holder: String] Jackie                   |   | [holder: String] Max                 |
    | [descript: String] iOS Developing Seminar |   | [descript: String] A quick brown fox |
    | [date: String] July 4th, 2016             |   | [date: String] February 30th, 2017   |
    |-------------------------------------------|   |--------------------------------------|

It is possible to store instantiations of more than one entity in one `NSManagedObject` array. For example, if we have another entity `Course` with one object, it can be stored in the previous array:  

    |--------[Entity: Activity]-----------------|   |--------[Entity: Activity]------------|    |--------[Entity: Course]--------|
    | [holder: String] Jackie                   |   | [holder: String] Max                 |    | [name: String] COM SCI 111     |
    | [descript: String] iOS Developing Seminar |   | [descript: String] A quick brown fox |    | [numOfStudents: Int] 100       |
    | [date: String] July 4th, 2016             |   | [date: String] February 30th, 2017   |    |--------------------------------|
    |-------------------------------------------|   |--------------------------------------|

### Sample Codes
Continue with our example about activities. Store new activities to Core Data:  

```swift
var activities = [NSManagedObject]();

func saveActivities(holder who: String, description what: String, date when: String) {
	// define App Delegates
	let appDelegate = UIApplication.sharedApplication().delegate as! AppDelegate
	let managedContext = appDelegate.managedObjectContext

	// instantiation of an object with entity "Activity"
	let entity = NSEntityDescription.entityForName("Activity", inManagedObjectContext: managedContext)
	let activity = NSManagedObject(entity: entity!, insertIntoManagedObjectContext: managedContext)

	// set values for the activity
	activity.setValue(who, forKey: "holder")
	activity.setValue(what, forKey: "descript")
	activity.setValue(when, forKey: "date")

	// commit unsaved changes to Core Data; add this new object to the array
	do {
		try managedContext.save()
		activities.append(activity)
	} catch let error as NSError {
		print("Could not save activity. \(error), \(error.userInfo)")
	}
}
```

Fetch all objects of entity `Activity` from Core Data to memory (to the `NSManagedObject` array):  

```swift
func fetchActivites() {
    // define App Delegates
    let appDelegate = UIApplication.sharedApplication().delegate as! AppDelegate
    let managedContext = appDelegate.managedObjectContext

    // create fetch request for an entity
    let fetchRequest = NSFetchRequest(entityName: "Activity")

    // fetch the objects and store them in the array
    do {
        let results = try managedContext.executeFetchRequest(fetchRequest)
        activities = results as! [NSManagedObject]
    } catch let error as NSError {
        print("Could not fetch activities. \(error), \(error.userInfo)")
    }
}
```

Display an object in a `NSManagedObject` array to UITableView: 
 
```swift
override func tableView(tableView: UITableView, cellForRowAtIndexPath indexPath: NSIndexPath) -> UITableViewCell {
    ... // get UITableViewCell
    let activity = activities[indexPath.row]
    cell.holderLabel.text = activity.valueForKey("holder") as? String
    cell.descriptionLabel.text = activity.valueForKey("descript") as? String
    cell.dateLabel.text = activity.valueForKey("date") as? String
}
```

When user deletes a cell in UITableView, delete it from both the `NSManagedObject` array and Core Data:  

```swift
override func tableView(tableView: UITableView, commitEditingStyle editingStyle: UITableViewCellEditingStyle, forRowAtIndexPath indexPath: NSIndexPath) {
    if editingStyle == .Delete {
        // Delete the row from the data source
        let appDelegate = UIApplication.sharedApplication().delegate as! AppDelegate

        appDelegate.managedObjectContext.deleteObject(activities[indexPath.row]) // delete from Core Data
        do {
            try appDelegate.managedObjectContext.save()
            activities.removeAtIndex(indexPath.row) // delete from "activities" array
            tableView.deleteRowsAtIndexPaths([indexPath], withRowAnimation: .Fade)
        } catch let error as NSError {
            print("Could not delete the activity. \(error), \(error.userInfo)")
        }

    } ...
}
```

### Future  
I am continue exploring how to edit an existing object in Core Data. The editing should be both reflected in the memory-based array of `NSManagedObject` and Core Data database.  

I will end this post with a graph I found pretty useful regarding functions of View Controllers.
![View Controller Graph](/assets/IGHA7.jpg "View Controller Graph")
