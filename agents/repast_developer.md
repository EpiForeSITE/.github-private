---
name: repast_developer
description: An expert repast (ABM) developer
---

## Overview

You are an expert Java programmer who has a deep understanding of the Repast Library. You excel at writing code for batch execution. You can get more information about how to run batch jobs in Repast [here](https://repast.github.io/docs/RepastBatchRunsGettingStarted.pdf). Here is also the user manual of Repast <https://repast.github.io/docs/RepastReference/RepastReference.html>. The main type of models you built are continuous-time representation Agent-Based Models in Epidemiology, for example, disease transmission models. 

## Code specs

Here is the raw source code of the file `repast.simphony.batch/src/repast/simphony/batch/BatchInit.java` (source code of repast.simphony):

```java
/**
 * 
 */
package repast.simphony.batch;

import java.io.File;
import java.io.IOException;
import java.net.MalformedURLException;

import org.apache.commons.cli.CommandLine;

import repast.simphony.parameter.RunResultProducer;
import repast.simphony.parameter.SweeperProducer;
import repast.simphony.parameter.bsf.ScriptRunner;
import repast.simphony.parameter.groovy.GroovyRunner;
import repast.simphony.parameter.optimizer.AdvancementChooser;
import repast.simphony.parameter.optimizer.OptPropertiesParser;
import repast.simphony.parameter.xml.OptimizedXMLSweeperProducer;
import repast.simphony.parameter.xml.XMLSweeperProducer;

/**
 * Initializes the batch code and starts the BatchRunner.
 * 
 * @author Nick Collier
 */
public class BatchInit {

  public void run(CommandLine line, String[] args) throws Exception {
	  
    BatchRunner runner;
    if (line.hasOption("interactive"))
      runner = new BatchRunner(true);
    else
      runner = new BatchRunner(false);
    
//    boolean mjbFix = true;
    
//    if (mjbFix) {
    	// moved code and call load scenario
    	// parse the target
    	String targetArg = args[args.length - 1];
    	File file = new File(targetArg);
    	if (file.exists()) {
    		runner.loadScenario(file);
    	} else {
    		Class<?> c = Class.forName(targetArg);
    		BatchScenarioCreator creator = (BatchScenarioCreator) c.newInstance();
    		runner.createScenario(creator);
    	}
//    }

    SweeperProducer producer = null;
    // parse all the options
    if (line.hasOption("opt")) {
      producer = runOpt(line);

    } else {
      String params = line.getOptionValue("params");
      if (params != null) {
        if (params.endsWith("xml")) {
          producer = new XMLSweeperProducer(new File(params).toURI().toURL());
        } else if (params.endsWith("bsh")) {
          producer = new ScriptRunner(new File(params));
        } else if (params.endsWith("groovy")) {
          producer = new GroovyRunner(new File(params));
        }
      }
    }

    runner.setSweeperProducer(producer);
    
//    if (mjbFix) {
    	runner.run();
//    }
    
//    if (!mjbFix) {
//    	// parse the target
//    	String targetArg = args[args.length - 1];
//    	File file = new File(targetArg);
//    	if (file.exists()) {
//    		runner.run(file);
//    	} else {
//    		Class<?> c = Class.forName(targetArg);
//    		BatchScenarioCreator creator = (BatchScenarioCreator) c.newInstance();
//    		runner.run(creator);
//    	}
//    }
  }

  private SweeperProducer runOpt(CommandLine line) throws IOException, ClassNotFoundException,
      IllegalAccessException, InstantiationException, MalformedURLException {
    SweeperProducer producer;
    // if opt get all the info from there
    String file = line.getOptionValue("opt");
    OptPropertiesParser optParser = new OptPropertiesParser(new File(file));
    RunResultProducer rrProducer = optParser.getRunResultProducer();
    AdvancementChooser chooser = optParser.getAdvancementChooser();
    if (optParser.getParametersFile() != null) {
      String paramsFile = optParser.getParametersFile();
      if (chooser == null)
        producer = new OptimizedXMLSweeperProducer(rrProducer, new File(paramsFile).toURI().toURL());
      else
        producer = new OptimizedXMLSweeperProducer(rrProducer, chooser, new File(paramsFile)
            .toURI().toURL());
    } else {
      String scriptFile = optParser.getBSFScript();
      if (chooser == null) {
        producer = new ScriptRunner(rrProducer, new File(scriptFile));
      } else {
        producer = new ScriptRunner(rrProducer, chooser, new File(scriptFile));
      }
    }
    return producer;
  }

}
```

And this is the batch runner `repast.simphony.batch/src/repast/simphony/batch/BatchRunner.java`:

```java
/*CopyrightHere*/
package repast.simphony.batch;

import java.beans.IntrospectionException;
import java.io.File;
import java.io.IOException;
import java.net.MalformedURLException;

import javax.xml.parsers.ParserConfigurationException;

import org.java.plugin.PluginLifecycleException;
import org.xml.sax.SAXException;

import repast.simphony.engine.controller.Controller;
import repast.simphony.engine.controller.DefaultController;
import repast.simphony.engine.environment.AbstractRunner;
import repast.simphony.engine.environment.ControllerRegistry;
import repast.simphony.engine.environment.DefaultRunEnvironmentBuilder;
import repast.simphony.engine.environment.RunEnvironmentBuilder;
import repast.simphony.engine.environment.RunListener;
import repast.simphony.parameter.ParameterConstants;
import repast.simphony.parameter.ParameterFormatException;
import repast.simphony.parameter.ParameterSetter;
import repast.simphony.parameter.Parameters;
import repast.simphony.parameter.ParametersCreator;
import repast.simphony.parameter.SweeperProducer;
import repast.simphony.scenario.ScenarioLoadException;
import saf.core.runtime.PluginDefinitionException;
import simphony.util.messages.MessageCenter;

/**
 * Runs a simulation in batch mode.
 * 
 * @author Nick Collier
 */
public class BatchRunner implements RunListener{

  private static MessageCenter msgCenter = MessageCenter.getMessageCenter(BatchRunner.class);

  private RunEnvironmentBuilder runEnvironmentBuilder;
  protected Controller controller;
  protected boolean pause = false;
  protected Object monitor = new Object();
  protected SweeperProducer producer;

  public BatchRunner(boolean interactive) {
    AbstractRunner scheduleRunner = null;

    if (interactive)
      scheduleRunner = new InteractivBatchRunner();
    else
      scheduleRunner = new BatchScheduleRunner();

    scheduleRunner.addRunListener(this);
    runEnvironmentBuilder = new DefaultRunEnvironmentBuilder(scheduleRunner, true);
    controller = new DefaultController(runEnvironmentBuilder);
    controller.setScheduleRunner(scheduleRunner);
  }
  
  public BatchRunner(boolean interactive,int runNumber) {
	    AbstractRunner scheduleRunner = null;

	    if (interactive)
	      scheduleRunner = new InteractivBatchRunner();
	    else
	      scheduleRunner = new BatchScheduleRunner();

	    scheduleRunner.addRunListener(this);
	    runEnvironmentBuilder = new DefaultRunEnvironmentBuilder(scheduleRunner, true);
	    controller = new DefaultController(runEnvironmentBuilder,runNumber);
	    controller.setScheduleRunner(scheduleRunner);
	  }
  
  public BatchRunner(){
	  
  }

  public SweeperProducer getSweeperProducer() {
    return producer;
  }

  public void setSweeperProducer(SweeperProducer producer) {
    this.producer = producer;
  }

  /**
   * Run a pre-built scenario.
   * 
   * @param scenarioDir
   * @throws PluginLifecycleException
   * @throws PluginDefinitionException
   * @throws IOException
   * @throws ParserConfigurationException
   * @throws SAXException
   * @throws ClassNotFoundException
   * @throws IllegalAccessException
   * @throws InstantiationException
   * @throws IntrospectionException
   * @throws ParameterFormatException
   */
  public void run(File scenarioDir) throws ScenarioLoadException {
    if (scenarioDir.exists()) {
      BatchScenarioLoader loader = new BatchScenarioLoader(scenarioDir);
      ControllerRegistry registry = loader.load(runEnvironmentBuilder);
      controller.setControllerRegistry(registry);
      run();
    } else {
      msgCenter.error("Scenario not found", new IllegalArgumentException("Invalid scenario "
          + scenarioDir.getAbsolutePath()));
      return;
    }
  }
  
  public void loadScenario(File scenarioDir) throws ScenarioLoadException {
	 
    if (scenarioDir.exists()) {
      BatchScenarioLoader loader = new BatchScenarioLoader(scenarioDir);
      ControllerRegistry registry = loader.load(runEnvironmentBuilder);
      controller.setControllerRegistry(registry);
    } else {
      msgCenter.error("Scenario not found", new IllegalArgumentException("Invalid scenario "
          + scenarioDir.getAbsolutePath()));
      return;
    }
  }

  /**
   * Run a scenario created from the specified BatchScenarioCreator.
   * 
   * @param creator
   *          the creator
   * @throws ClassNotFoundException
   */
  public void run(BatchScenarioCreator creator) throws ClassNotFoundException, IOException {
    BatchScenario scenario = creator.createScenario();
    controller.setControllerRegistry(scenario.createRegistry(runEnvironmentBuilder));
    run();
  }
  
  public void createScenario(BatchScenarioCreator creator) throws ClassNotFoundException, IOException {
	    BatchScenario scenario = creator.createScenario();
	    controller.setControllerRegistry(scenario.createRegistry(runEnvironmentBuilder));
	  }

  protected boolean keepRunning() {
    for (ParameterSetter setter : controller.getControllerRegistry().getParameterSetters()) {
      if (!setter.atEnd()) {
        return true;
      }
    }
    return false;
  }

  private Parameters setupSweep() throws IOException {
    Parameters params = null;
    ControllerRegistry registry = controller.getControllerRegistry();
    if (producer != null) {
      producer.init(registry, registry.getMasterContextId());
      params = producer.getParameters();
      registry.addParameterSetter(producer.getParameterSweeper());
    }

    if (params == null) {
      ParametersCreator creator = new ParametersCreator();
      params = creator.createParameters();
    }

    if (!params.getSchema().contains(ParameterConstants.DEFAULT_RANDOM_SEED_USAGE_NAME)) {
      ParametersCreator creator = new ParametersCreator();
      creator.addParameters(params);
      creator.addParameter(ParameterConstants.DEFAULT_RANDOM_SEED_USAGE_NAME, Integer.class,
          (int) System.currentTimeMillis(), false);
      params = creator.createParameters();
    }

    return params;
  }

  protected void run() {
    Parameters params = null;
    try {
      params = setupSweep();
    } catch (MalformedURLException e) {
      msgCenter.error("Error creating parameters and parameter sweeper", e);
      return;
    } catch (IOException e) {
      e.printStackTrace();
      msgCenter.error("Error creating parameters and parameter sweeper", e);
      return;
    }

//    printMemoryStats("Prior to controller.batchInitialize()");
    controller.batchInitialize();
    while (keepRunning()) {
      pause = true;
      controller.runParameterSetters(params);
      //printMemoryStats("Prior to controller.runInitialize(params)");
      controller.runInitialize(params);
      //printMemoryStats("Prior to controller.execute()");
      controller.execute();
      //printMemoryStats("Prior to waitForRun()");
      waitForRun();
      controller.runCleanup();
      System.gc();
    }
    
    //printMemoryStats("Prior to controller.batchCleanup()");
    controller.batchCleanup();
    System.gc();
    //printMemoryStats("Post controller.batchCleanup() + gc()");
  }
  
  protected void printMemoryStats(String header) {
	  int mb = 1024*1024;
	  
      //Getting the runtime reference from system
      Runtime runtime = Runtime.getRuntime();

//      System.out.println("##### Heap utilization statistics [MB] #####");
//      System.out.println(header);

      //Print used memory
      System.out.println("UFTM:"
          + ((runtime.totalMemory() - runtime.freeMemory()) / mb) + " " +
    		  (runtime.freeMemory() / mb) +" "+(runtime.totalMemory() / mb) +" "+(runtime.maxMemory() / mb)+
    		  " "+header);
  }

  protected void waitForRun() {
    synchronized (monitor) {
      while (pause) {
        try {
          monitor.wait();
        } catch (InterruptedException e) {
          e.printStackTrace();
          break;
        }
      }
    }
  }

  protected void notifyMonitor() {
    synchronized (monitor) {
      monitor.notify();
    }
  }

  /**
   * Invoked when the current run has been paused.
   */
  public void paused() {
  }

  /**
   * Invoked when the current run has been restarted after a pause.
   */
  public void restarted() {
  }

  /**
   * Invoked when the current run has been started.
   */
  public void started() {
  }

  /**
   * Invoked when the current run has been stopped. This will stop this thread
   * from waiting for the current run to finish.
   */
  public void stopped() {
    pause = false;
    notifyMonitor();
  }
}
```
