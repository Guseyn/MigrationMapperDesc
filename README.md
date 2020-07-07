# MigrationMapperDesc

Here I just explain how Migration Mapper(https://github.com/Guseyn/MigrationMapper) works, at least as I understand from the code/readme and the output of the program.

## Steps in General Logic

### 1. Collection of commits from specified git repos.

Here we clone all specified in the folder `/MigrationMapper/MigrationMapper/data/gitRepositories.csv` repositories with all theirs commits. We collect all commits into the file `/MigrationMapper/MigrationMapper/Clone/Process/app_commits.txt`, also we save all diffs via git patches in the folder: `/MigrationMapper/MigrationMapper/Clone/Diffs`(**TODO: //investigate these folder in more details later**)

Also we collect all commits into database:

```
mysql> describe AppCommits;
+---------------+--------------+------+-----+---------+-------+
| Field         | Type         | Null | Key | Default | Extra |
+---------------+--------------+------+-----+---------+-------+
| AppID         | int(11)      | YES  |     | NULL    |       |
| CommitID      | varchar(200) | YES  |     | NULL    |       |
| CommitDate    | datetime     | YES  |     | NULL    |       |
| DeveloperName | varchar(100) | YES  |     | NULL    |       |
| CommitText    | text         | YES  |     | NULL    |       |
+---------------+--------------+------+-----+---------+-------+
```

### 2. Find migration rules

First, we select all already existing project libs from the table:

```
mysql> describe ProjectLibrariesView;
+-------------+--------------+------+-----+---------+-------+
| Field       | Type         | Null | Key | Default | Extra |
+-------------+--------------+------+-----+---------+-------+
| ProjectsID  | int(11)      | YES  |     | NULL    |       |
| CommitID    | varchar(200) | YES  |     | NULL    |       |
| LibraryName | varchar(200) | YES  |     | NULL    |       |
| isAdded     | int(11)      | YES  |     | NULL    |       |
| PomPath     | varchar(200) | YES  |     | NULL    |       |
+-------------+--------------+------+-----+---------+-------+
```

Then we find added and removed libraries from the commits by using Cartesian Product. Then we filter Cartesian Product by some logic which I cannot understand due to porr naming of the variables. And only then we looking for migration rules by such parameters like `frequency, numberOfTrueMigrationRules, numberOfTrueUpgradeRules, numberOfFalseRules`(needs to be explored more). Then we save all migration rules into db:

```
mysql> describe MigrationRules;
+-------------+--------------+------+-----+---------+----------------+
| Field       | Type         | Null | Key | Default | Extra          |
+-------------+--------------+------+-----+---------+----------------+
| ID          | int(11)      | NO   | PRI | NULL    | auto_increment |
| FromLibrary | varchar(200) | YES  |     | NULL    |                |
| ToLibrary   | varchar(200) | YES  |     | NULL    |                |
| Frequency   | int(11)      | YES  |     | NULL    |                |
| Accuracy    | double       | YES  |     | NULL    |                |
| isVaild     | int(11)      | YES  |     | 0       |                |
+-------------+--------------+------+-----+---------+----------------+
```

### 3. Detecting code fragments for migrations

Here we search for migration using library changes in pom file and somehow we add code segments of megration into db:

```
mysql> describe MigrationSegments;
+-----------------+--------------+------+-----+---------+-------+
| Field           | Type         | Null | Key | Default | Extra |
+-----------------+--------------+------+-----+---------+-------+
| MigrationRuleID | int(11)      | YES  |     | NULL    |       |
| AppID           | int(11)      | YES  |     | NULL    |       |
| CommitID        | varchar(200) | YES  |     | NULL    |       |
| FromCode        | text         | YES  |     | NULL    |       |
| ToCode          | text         | YES  |     | NULL    |       |
| fileName        | text         | YES  |     | NULL    |       |
| fromLibVersion  | text         | YES  |     | NULL    |       |
| toLibVersion    | text         | YES  |     | NULL    |       |
+-----------------+--------------+------+-----+---------+-------+
```

### 4. Docs 

Here we just collecting and generating docs to `Docs` folder.

### 5. Printing fragments as HTML pages

### 6. Apply substitution algorithm ==> I need more time to figure how it work.

### 7. Printing Mapping reeesults as HTML.

## How to work with database entities

```
//Return list of migrations between two pairs of libraries( added/removed)
LinkedList<MigrationRule> migrationRules= new MigrationRuleDB().getMigrationRulesWithoutVersion(1);

for (MigrationRule migrationRule : migrationRules) {
 System.out.println("== Migration Rule "+ migrationRule.FromLibrary +
      " <==> "+  migrationRule.ToLibrary +"==");

 /*
 *  For every migrations, retrieve list of detected Method mapping
 *  Between Two APIs
 */
 ArrayList<Segment> segmentList = new MigrationMappingDB().getFunctionMapping(String.valueOf(migrationRule.ID), false, false);

 for (Segment segment : segmentList) {

  segment.print();

  // Print all removed method signatures With Docs
  printMethodWithDocs( migrationRule.FromLibrary,segment.removedCode);  

  // Print all added method signatures With Docs
  printMethodWithDocs( migrationRule.ToLibrary,segment.addedCode);

 } // End fragment for every migration

}  // End library migration


/* 
* This method takes list of methods signatures with library that methods belong to.
* It will print the signatures and Docs for every method
*/
void printMethodWithDocs(String libraryName,ArrayList<String> listOfMethods ) {

 // For every add method print the Docs
 for(String methodSignature: listOfMethods){

  // Convert  method signatures as String to Object
  MethodObj methodFormObj= MethodObj.GenerateSignature(methodSignature);

  //retrieve Docs from the library for method has that name
  ArrayList<MethodDocs>  toLibrary = new LibraryDocumentationDB()
                                           .getDocs( libraryName,methodFormObj.methodName);

  //Map method signatures to docs
  MethodDocs methodFromDocs = MethodDocs.GetObjDocs(toLibrary, methodFormObj);

  if(methodFromDocs.methodObj== null) {
   System.err.println("Cannot find Docs for: "+ methodSignature);
   continue;
  }
  methodFromDocs.print();      
 }
}
```

## Conslusion about applying patters.

As it seems to me most probably this library just creats only snippets with code before and after via simple strings as I don't see how substitution algorithm works, as we need to apply migrations on some code on input, but I don't see that it work this way. But we can definitely use such code segments, parse them and creeate our own algorithm.

## Can we use project database?

Actually I have not found dataabase with ready migration tools, I guess we can only generate them and then create an algorithm to apply them.

## Stack of technologies(project deps)

1. com.github.javaparser - it's in pom, but I don't see any usage of it in the code, which is weird.
2. jdom (To provide a complete, Java-based solution for accessing, manipulating, and outputting XML data from Java code.)
3. jsoup is a Java library for working with real-world HTML. It provides a very convenient API for fetching URLs and extracting and manipulating data, using the best of HTML5 DOM methods and CSS selectors.
4. mysql-connector-java - for working with MySQL
5. org.json - json parser (only in the pom, but never used)
6. java-string-similarity -  calculates a normalized distance or similarity score between two strings. A score of 0.0 means that the two strings are absolutely dissimilar, and 1.0 means that absolutely similar (or equal). Anything in between indicates how similar each the two strings are.
7. commons-io - just set of usefult util methods

