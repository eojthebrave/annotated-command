# Consolidation\AnnotationCommand

Initialize Symfony Console commands from annotated command class methods.

[![Travis CI](https://travis-ci.org/consolidation-org/annotation-command.svg?branch=master)](https://travis-ci.org/consolidation-org/annotation-command) [![Scrutinizer Code Quality](https://scrutinizer-ci.com/g/consolidation-org/annotation-command/badges/quality-score.png?b=master)](https://scrutinizer-ci.com/g/consolidation-org/annotation-command/?branch=master) [![License](https://poser.pugx.org/consolidation/annotation-command/license)](https://packagist.org/packages/consolidation/annotation-command)

## Component Status

Currently in use in [Robo](https://github.com/Codegyre/Robo).

## Motivation

Symfony Console provides a set of classes that are widely used to implement command line tools. Increasingly, it is becoming popular to use annotations to describe the characteristics of the command (e.g. its arguments, options and so on) implemented by the annotated method.

Extant commandline tools that utilize this technique include:

- [Robo](https://github.com/codegyre/robo)
- [wp-cli](https://github.com/wp-cli/wp-cli)
- [Pantheon Terminus](https://github.com/pantheon-systems/terminus)

This library provides routines to produce the Symfony\Component\Console\Command\Command from all public methods defined in the provided class.

## Example Annotated Command Class
The public methods of the command class define its commands, and the parameters of each method define its arguments and options. The command options, if any, are declared as the last parameter of the methods. The options will be passed in as an associative array; the default options of the last parameter should list the options recognized by the command.

The rest of the parameters are arguments. Parameters with a default value are optional; those without a default value are required.
```
class MyCommandClass
{
    /**
     * This is the my:cat command
     *
     * This command will concatinate two parameters. If the --flip flag
     * is provided, then the result is the concatination of two and one.
     *
     * @param integer $one The first parameter.
     * @param integer $two The other parameter.
     * @option $flip Whether or not the second parameter should come first in the result.
     * @aliases c
     * @usage bet alpha --flip
     *   Concatinate "alpha" and "bet".
     */
    public function myCat($one, $two, $options = ['flip' => false])
    {
        if ($options['flip']) {
            return "{$two}{$one}";
        }
        return "{$one}{$two}";
    }
}
``` 
## Hooks

Commandfiles may provide hooks in addition to commands. A commandfile method that contains a @hook annotation is registered as a hook instead of a command.  The format of the hook annotation is:
```
@hook type commandname
```
The commandname may be the command's primary name (e.g. `my:command`), it's method name (e.g. myCommand) or any of its aliases.

There are four types of hooks supported:

- Validate
- Alter
- Status
- Extract

Each of these also have "pre" and "post" varieties, to give more flexibility vis-a-vis hook ordering (and for consistency). Note that many validate and alter hooks may run, but the first status or extract hook that successfully returns a result will halt processing of further hooks of the same type.

### Validate Hook

Validation hooks examine the arguments and options passed to a command. A validation hook may take one of several actions:

- Do nothing. This indicates that validation succeeded.
- Return an array. This alters the arguments that will be used during command execution.
- Return a CommandError. Validation fails, and execution stops. The CommandError contains a status result code and a message, which is printed.
- Throw an exception. The exception is converted into a CommandError.

Any number of validation hooks may run, but if any fails, then execution of the command stops.

### Alter Hook

An alter hook changes the result object. Alter hooks should only operate on result objects of a type they explicitly recognize. They may return an object of the same type, or they may convert the object to some other type.

If something goes wrong, and the alter hooks wishes to force the command to fail, then it may either return a CommandError object, or throw an exception.

### Status Hook

The result object returned by a command may be a compound object that contains multiple bits of information about the command result.  If the result object implements ExitCodeInterface, then the `getExitCode()` method of the result object is called to determine what the status result code for the command should be. If ExitCodeInterface is not implemented, then all of the status hooks attached to this command are executed; the first one that successfully returns a result will stop further execution of status hooks, and the result it returned will be used as the status result code for this operation.

If no status hook returns any result, then success is presumed.

### Extract Hook

The result object returned by a command may be a compound object that contains multiple bits of information about the command result.  If the result object implements OutputDataInterface, then the `getOutputData()` method of the result object is called to determine what information should be displayed to the user as a result of the command's execution. If ExitCodeInterface is not implemented, then all of the extract hooks attached to this command are executed; the first one that successfully returns output data will stop further execution of extract hooks.

If no extract hook returns any data, then the result object itself is printed if it is a string; otherwise, no output is emitted (other than any produced by the command itself).

## Output

If a command method returns an integer, it is used as the command exit status code. If the command method returns a string, it is printed.

If the [Consolidation/OutputFormatters](https://github.com/consolidation-org/output-formatters) project is used, then users may specify a --format option to select the formatter to use to transform the output from whatever form the command provides to a string.  To make this work, the application must provide a formatter to the AnnotationCommandFactory.  See [API Usage](#api-usage) below.

## Logging

The Annotation-Command project is completely agnostic to logging. If a command wishes to log progress, then the CommandFile class should implement LoggerAwareInterface, and the Commandline tool should inject a logger for its use via the LoggerAwareTrait `setLogger()` method.  Using [Robo](https://github.com/codegyre/robo) is recommended.

## Access to Symfony Objects

If you want access to the Symfony Command, e.g. to get a reference to the helpers in order to call some legacy code, simply typehint the first parameter of your command method as a \Symfony\Component\Console\Command\Command, and the command object will be passed in. The other parameters define your commands arguments and options, as usual.
```
class MyCommandClass
{
    public function testCommand(Command $command, $message)
    {
        $formatter = $command->getHelperSet()->get('formatter');
        return $formatter->formatSection('test', $message);
    }
}
```
Similarly, references to the $input and $output objects are passed in if any of the initial parameters to the method are of type InputInterface or OutputInterface, respectively.

## API Usage

To use annotated commands in an application, pass an instance of your command class in to AnnotationCommandFactory::createCommandsFromClass(). The result will be a list of Commands that may be added to your application.
```
$myCommandClassInstance = new MyCommandClass();
$commandFactory = new AnnotationCommandFactory();
$commandFactory->commandProcessor()->setFormatterManager(new FormatterManager());
$commandList = $commandFactory->createCommandsFromClass($myCommandClassInstance);
foreach ($commandList as $command) {
    $application->add($command);
}
```
You may have more than one command class, if you wish. If so, simply call AnnotationCommandFactory::createCommandsFromClass() multiple times.

Note that the `setFormatterManager()` operation is optional; omit this if not using [Consolidation/OutputFormatters](https://github.com/consolidation-org/output-formatters).

## Comparison to Existing Solutions

The existing solutions used their own hand-rolled regex-based parsers to process the contents of the DocBlock comments. consolidation/annotation-command uses the phpdocumentor/reflection-docblock project (which is itsle a regex-based parser) to interpret DocBlock contents. 

## Caution Regarding Dependency Versions

Note that phpunit requires phpspec/prophecy, which in turn requires phpdocumentor/reflection-docblock version 2.x.  This blocks consolidation/annotation-command from using the 3.x version of reflection-docblock. When prophecy updates to a newer version of reflection-docblock, then annotation-command will be forced to follow (or pin to an older version of phpunit). The internal classes of reflection-docblock are not exposed to users of consolidation/annotation-command, though, so this upgrade should not affect clients of this project.
