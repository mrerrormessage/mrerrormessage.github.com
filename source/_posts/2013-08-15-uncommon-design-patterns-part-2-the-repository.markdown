---
layout: post
title: "uncommon design patterns part 2 - the repository"
date: 2013-08-16 20:55
comments: true
published: false
categories: 
---

Pattern Two: The Repository

    module Directory
      class FileRepository
        def initialize(filesystem)
          @filesystem = filesystem
        end

        def update(file)
          @filesystem.write(file)
        end 
      end
    end

What is it?
A repository is all about persisting an object.
Persistence is a messy problem, but having the object to be persisted deal with it creates more problems than it is worth.
In an upcoming blog post I hope to talk about why I believe ActiveRecord is an anti-pattern.
In the meantime
