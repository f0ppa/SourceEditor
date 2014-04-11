SourceEditor
============

Library to tokenize, edit and override PHP classes.

Installation
============
It can be installed with [composer](https://getcomposer.org/doc/00-intro.md)  in the CLI.
First add the package to your composer.json
```json
    "require": {
       ...
        "docdigital/php-class-editor": "dev-master"
    },
```

then update your project

```
$ php composer.phar update docdigital/php-class-editor
```

And including the classes in your code:

```php
use DocDigital\Lib\SourceEditor\PhpClassEditor;
use DocDigital\Lib\SourceEditor\ClassStructure\ClassElement;
```
You might also need to include composer's [autoload](https://getcomposer.org/doc/04-schema.md#psr-0)

Overview
========

Attempts to handle modiffications in existing PHP Class, by adding annotations
 to specific properties, adding properties and methods.
 
 This is useful for instance to add annotations to an entity Generated by 
 {@link \Doctrine\ORM\Tools\EntityGenerator}
 
 The Idea is to make is simple to get parts of the code as relevant elements and 
 manipulate them. In the next example I have 3 relevant elements, each having  
 sub-elements accessible from the main (parent??) one.
 
```php
 /**
  * Class Doc Block
  */
 class a
 {
     /**
      * Attr Doc Block
      */
     public $attr
     
     /**
      * Fn Doc Block
      */
     public function b()
     {
   }
 }
```
```XML
 <elemnet class>
     <docBlock/>
     <element attribute>
         <docBlock/>
     </element >
     <element method>
         <docBlock/>
     </element >
 </elemnet> 

```
 Then you should be able to issue: 
```php
    /* @var $classEditor DocDigital\Lib\SourceEditor\PhpClassEditor */
    $classEditor->parseFile($classPath);
    $classEditor->getClass('a')->getMethod('b')->addAnnotation('@auth Juan Manuel Fernandez <juanmf@gmail.com>');
    $classEditor->getClass('a')->getAttribute('attr')->addAnnotation('@Assert\Choice(...)');
    $classEditor->getClass('a')->addAttribute($attr2);
    $classEditor->getClass('a')->addUse('use DocDigital\Bundle\DocumentBundle\DocumentGenerator\Annotation as DdMapping;');
    $classEditor->getClass('a')->addConst('    const CONSTANT = 1;');
```

When finished you must render the file.
```php
    // Element::render() is a composite pattern implementation, class renders its children and so on. 
    file_put_contents('PATH/TO/CLASS/FILE', $classEditor->getClass('a')->render(false));
```

to get a reference to the Class representation (ClassElement) you have to do something like this

```php
    /**
     * Used to add annotations and validaton methods on generated php classes
     * 
     * @var PhpClassEditor 
     */
    private $phpClassEditor;
    
    /**
     * Parses The just writteng PHP Entity and returns a CalssElement representation.
     * 
     * @param \DocDigital\Bundle\DocumentBundle\Entity\DocumentType $docDefinition
     * 
     * @return DocDigital\Lib\SourceEditor\ClassStructure\ClassElement A class 
     * definition representing the Document being Modiffied.
     */
    public function parseDocument(DocumentType $docDefinition)
    {
        $classPath = $this->getDocumentClassPath($docDefinition);
        $classes = $this->phpClassEditor->parseFile($classPath);
        $document = end($classes);
        return $document;
    }
```

In this example Method I add Constants to a Doctrine Entity programatically, 
to specify ROLES the user must have

```php
    /**
     * Adds standar ROLES to this entity, as constants useful to ask consistently
     * for a given ROLE in any Controller or view.
     * 
     * @param ClassElement $docDefinition
     * @param DocumentType $docDefinition
     */
    public function addEntityRoles(ClassElement $document, DocumentType $docDefinition)
    {
        $docBlock = "/**\n    * Role required for this Document to perform this action\n    */";
        $document->addConst(
                sprintf('    const ROLE_LIST = \'ROLE_%s_LIST\';', strtoupper($docDefinition->getClassName())),
                $docBlock
            );
        $document->addConst(sprintf('    const ROLE_VIEW = \'ROLE_%s_VIEW\';', strtoupper($docDefinition->getClassName())));
        $document->addConst(sprintf('    const ROLE_EDIT = \'ROLE_%s_EDIT\';', strtoupper($docDefinition->getClassName())));
        $document->addConst(sprintf('    const ROLE_CREATE = \'ROLE_%s_CREATE\';', strtoupper($docDefinition->getClassName())));
        $document->addConst(sprintf('    const ROLE_DELETE = \'ROLE_%s_DELETE\';', strtoupper($docDefinition->getClassName())));
    }
```

the result afer overriding the file is:

```php
/**
 * Asd
 *
 * @ORM\Table(name="custom_doc_asd")
 * @ORM\Entity
 */
class Asd extends \DocDigital\Bundle\DocumentBundle\Entity\Document
{

    /**
    * Role required for this Document to perform this action
    */
    const ROLE_LIST = 'ROLE_ASD_LIST';

    /**
     *
     */
    const ROLE_VIEW = 'ROLE_ASD_VIEW';

    /**
     *
     */
    const ROLE_EDIT = 'ROLE_ASD_EDIT';

    /**
     *
     */
    const ROLE_CREATE = 'ROLE_ASD_CREATE';

    /**
     *
     */
    const ROLE_DELETE = 'ROLE_ASD_DELETE';
    ...
```

Internals
=========

Here's TokenParser.php's DocBlock. The parser is a decoupled piece of code, PhpClassEditor relies on it and gives it 2 Closures to arrange the class composite structure.

```php
/**
 * Handles Token classification, code {@link ElementBuilder} creation and code context/scope
 * changes detection. Each ElementBuilder will contain either a significant code part or gap code,
 * like T_WHITESPACE or unclassified code, like method body (as currently not inspecting inside method).
 * 
 * Every time an ElementBuilder is closed, or a context changes (which also closes an ElementBuilder)
 * it gets forwarded to a couple of callback closures given by client code:<pre>
 *    {@link self::$contextChangeClosure}
 *    {@link self::$processElementClosure}
 *</pre>
 * Which are passed as parameters to {@link self::setSource()}
 * 
 * The basic sequence is:<pre>
 * +------------+  +--------------+
 * | :ClientObj |  | :TokenParser |
 * +------------+  +--------------+
 *      |              |
 *      |--setSource-->|
 *      |--parseCode-->|--readToken--+[forEach Token, this iteration changes context as token is a contextStart]
 *      |              ||<-----------+
 *      |              ||--read<CONTEXT>Token--+
 *      |              |||<--------------------+
 *      |              |||--_checkContextChange--+
 *      |              ||||<---------------------+
 *      |              ||||--_isContextStart--+
 *      |              |||||<-----------------+
 *      |              ||||--_changeContext--+ [if Context changed e.g. class=>method]
 *      |              |||||<----------------+
 *      |              |||||--_startNewElement--+
 *      |              ||||||<------------------+
 *      |              ||||||------------------------------+
 *      ||<--$processElementClosure(self::elementBuilder)--+
 *      |              |||||
 *      |              |||||------------------------------------------------------------+
 *      ||<--$contextChangeClosure($newContext, $currentContext, self::elementBuilder)--+
 *      |              |||||
 *      |              |||||--readToken--+ [reReads Token without increasing {@link self::pointer}]
 *      |              ||||||<-----------+
 *      |              |
 *      |              |[continue Looping forEach Token at parseCode, now reads a token that doesn't change context]
 *      |              |
 *      |              |--readToken--+[forEach Token]
 *      |              ||<-----------+
 *      |              ||--read<CONTEXT>Token--+
 *      |              |||<--------------------+
 *      |              |||--_checkContextChange--+
 *      |              ||||<---------------------+
 *      |              ||||--_isContextStart--+
 *      |              |||||<-----------------+
 *      |              ||||--_isContextEnd--+ [Called only if _isContextStart returns false, also 
 *      |              |||||<---------------+      returns false for this token]
 *      |              |||
 *      |              |||--_loadTokenInElement--+ [context didn't change, add token to ElementBuilder]
 *      |              ||||<---------------------+
 *      |              ||||--_startNewElement--+   [Only if token is a delimiter Flag that closes current 
 *      |              |||||<------------------+       self::elementBuilder again calling $processElementClosure]
 *      |              |
 *      |              |[continue Looping forEach Token at parseCode, now reads a token that doesn't change context]
 *      |              |
 * 
 * @author Juan Manuel Fernandez <juanmf@gmail.com>
 */
 ```
