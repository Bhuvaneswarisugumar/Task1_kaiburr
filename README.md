# Java Shell Task Manager

This is a Spring Boot project to run shell tasks via REST API.

## Features
- Run shell commands
- Save task output to MongoDB
- RESTful endpoints

## Technologies Used
- Java 17
- Spring Boot
- MongoDB

## Getting Started
Run the project using:
./mvnw spring-boot:run

### folder name taskapp
First Task
### Task.java
package com.example.taskapp.model;
import java.util.List;

import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;

@Document(collection = "tasks")
public class Task extends Object{
    @Id
    private String id;
    private String name;
    private String owner;
    private String command;
    private List<TaskExecution> taskExecutions;
    public String getId() {
        return id;
    }
    public void setId(String id) {
        this.id = id;
    }
    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
    public String getOwner() {
        return owner;
    }
    public void setOwner(String owner) {
        this.owner = owner;
    }
    public String getCommand() {
        return command;
    }
    public void setCommand(String command) {
        this.command = command;
    }
    public List<TaskExecution> getTaskExecutions() {
        return taskExecutions;
    }
    public void setTaskExecutions(List<TaskExecution> taskExecutions) {
        this.taskExecutions = taskExecutions;
    }

    // Getters and setters
}

### TaskExecution.java
package com.example.taskapp.model;

import java.util.Date;

public class TaskExecution {
    private Date startTime;
    private Date endTime;
    private String output;
    public Date getStartTime() {
        return startTime;
    }
    public void setStartTime(Date startTime) {
        this.startTime = startTime;
    }
    public Date getEndTime() {
        return endTime;
    }
    public void setEndTime(Date endTime) {
        this.endTime = endTime;
    }
    public String getOutput() {
        return output;
    }
    public void setOutput(String output) {
        this.output = output;
    }

    // Getters and setters
}

### TaskService.java
package com.example.taskapp.service;
import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.util.ArrayList;
import java.util.Date;
import java.util.List;
import java.util.Optional;

import org.springframework.stereotype.Service;

import com.example.taskapp.model.Task;
import com.example.taskapp.model.TaskExecution;
import com.example.taskapp.repository.TaskRepository;

@Service
public class TaskService {
    private TaskRepository repo;

    public void setRepo(TaskRepository repo) {
        this.repo = repo;
    }
    public TaskService(TaskRepository repo) {
        this.repo = repo;
    }
    public List<Task> getAllTasks() {
        return repo.findAll();
    }
    public Optional<Task> getTaskById(String id) {
        return repo.findById(id);
    }
    public List<Task> getTasksByName(String name) {
        return repo.findByNameContaining(name);
    }
    public Task saveTask(Task task) throws IllegalArgumentException {
        if (!task.getCommand().startsWith("echo")) {
            throw new IllegalArgumentException("Unsafe command");
        }
        return repo.save(task);
    }
    public void deleteTask(String id) {
        repo.deleteById(id);
    }
    public Task addExecution(String id) throws Exception {
        Task task = repo.findById(id).orElseThrow(() -> new RuntimeException("Task not found"));
        Date start = new Date();
        Process process = Runtime.getRuntime().exec(task.getCommand());
        BufferedReader reader = new BufferedReader(new InputStreamReader(process.getInputStream()));
        StringBuilder output = new StringBuilder();
        String line;
        while ((line = reader.readLine()) != null) {
            output.append(line).append("\n");
        }
        process.waitFor();
        Date end = new Date();
        TaskExecution exec = new TaskExecution();
        exec.setStartTime(start);
        exec.setEndTime(end);
        exec.setOutput(output.toString());
        if (task.getTaskExecutions() == null)
            task.setTaskExecutions(new ArrayList<>());
        task.getTaskExecutions().add(exec);
        return repo.save(task);
    }
}

    

### TaskRepository.java
package com.example.taskapp.repository;

import java.util.List;

import org.springframework.data.mongodb.repository.MongoRepository;

import com.example.taskapp.model.Task;

public interface TaskRepository extends MongoRepository<Task, String> {
    List<Task> findByNameContaining(String name);
}

### TaskController.java
package com.example.demo.controller;

import com.example.demo.model.*;
import com.example.demo.repository.TaskRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.util.*;

@RestController
@RequestMapping("/tasks")
public class TaskController {

    @Autowired
    private TaskRepository repo;

    @GetMapping
    public Object getAllOrOne(@RequestParam(required = false) String id) {
        if (id == null) return repo.findAll();
        return repo.findById(id).orElseThrow(() -> new RuntimeException("Task not found"));
    }

    @PutMapping
    public Task createTask(@RequestBody Task task) {
        if (task.getCommand().contains("rm") || task.getCommand().contains("shutdown")) {
            throw new IllegalArgumentException("Unsafe command");
        }
        return repo.save(task);
    }

    @DeleteMapping
    public void deleteTask(@RequestParam String id) {
        repo.deleteById(id);
    }

    @GetMapping("/search")
    public List<Task> searchByName(@RequestParam String name) {
        List<Task> tasks = repo.findByNameContaining(name);
        if (tasks.isEmpty()) throw new RuntimeException("No tasks found");
        return tasks;
    }

    @PutMapping("/{id}/execute")
    public Task executeTask(@PathVariable String id) throws Exception {
        Task task = repo.findById(id).orElseThrow(() -> new RuntimeException("Task not found"));

        Date start = new Date();
        Process process = Runtime.getRuntime().exec(task.getCommand());
        BufferedReader reader = new BufferedReader(new InputStreamReader(process.getInputStream()));
        StringBuilder output = new StringBuilder();
        String line;
        while ((line = reader.readLine()) != null) output.append(line).append("\n");
        process.waitFor();
        Date end = new Date();

        TaskExecution exec = new TaskExecution();
        exec.setStartTime(start);
        exec.setEndTime(end);
        exec.setOutput(output.toString());

        if (task.getTaskExecutions() == null) task.setTaskExecutions(new ArrayList<>());
        task.getTaskExecutions().add(exec);

        return repo.save(task);
    }
}

### application.properties

spring.application.name=taskapp
spring.data.mongodb.uri=mongodb://localhost:27017/taskdb
server.port=8080


### settings.json
{
    
  "java.jdt.ls.java.home": "C:\\Program Files\\Java\\jdk-24",
  "containers.containerClient": "com.microsoft.visualstudio.containers.podman"

}

### output
![Postman Output](./images/postman.output.jpg)

This output shows following details

{
  "id": "123",
  "name": "Print Hello",
  "owner": "John Smith",
  "command": "echo Hello World"
}



