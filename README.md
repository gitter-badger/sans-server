sans-server
==============

SansServer is a small web application implemented with a server-less architecture. It uses Amazon API Gateway to expose the Lambda function as HTTP endpoints and uses Identity and Access Management (IAM) and Amazon Cognito to retrieve temporary credentials for a user and authorize access to its APIs with.  Amazon DynamoDB is used for the application's persistent storage system.

The goal of this project was to create a prototype that would allow you to work towards providing a cost-efficient solution that is scalable and highly available.  Please keep in mind, this project has been written as an example and therefore should not be considered production ready.  It has; however, tried to follow a few best practices to give you a fairly clean development framework to build on top of.

Features
--------

  * Java-based Lambda Functions
  * SAP Web application front-end with Unauthenticated and Authenticated pages
  * Project properties file used at runtime
  * Configuration schema to allow for the isolation of multiple deployments:  Multi-Engineer Development, QA, Production
  * JUnit test harness for locally invoking your Lambda functions before deployment
  
Demo Site
---------------
http://lambda.nfbsotware.com

Getting started
---------------
Coming soon

SansServer Framework
---------------
This project contains a number of java classes that try to help simplify the authoring of lambda functions, including a base handler to configure your system and collect your request parameters.  The base handler also defines a simple response format to allow the client web-app to determine the status of any given transaction.  Below is an example of a basic Lambda function that utilizes this framework to view a user record within the system:

```java
package com.nfbsoftware.sans_server.user.lambda;

import com.nfbsoftware.sans_server.core.lambda.BaseLambdaHandler;
import com.nfbsoftware.sans_server.core.model.HandlerResponse;
import com.nfbsoftware.sans_server.core.util.Entity;
import com.nfbsoftware.sans_server.core.util.StringUtil;
import com.nfbsoftware.sans_server.user.dao.UserDaoImpl;
import com.nfbsoftware.sans_server.user.model.User;

/**
 * The ViewUser function simply returns the user record requested
 * 
 * @author Brendan Clemenzi
 */
public class ViewUser extends BaseLambdaHandler
{
    /**
     * 
     * @return
     * @throws Exception
     */
    @Override
    public HandlerResponse processRequest() throws Exception
    {
        HandlerResponse handlerResponse = new HandlerResponse();
        
        try
        {
        	//  Get our request parameter
            String username = StringUtil.emptyIfNull(this.getParameter("username"));
            
            // Get out configuration properties
            String region = this.getProperty(Entity.FrameworkProperties.AWS_REGION);
            String accessKey = this.getProperty(Entity.FrameworkProperties.AWS_ACCESS_KEY);
            String secretKey = this.getProperty(Entity.FrameworkProperties.AWS_SECRET_KEY);
            String dynamoDbTableNamePrefix = this.getProperty(Entity.FrameworkProperties.AWS_DYNAMODB_TABLE_NAME_PREFIX);
            
            // Init out User DAO for talking with DynamoDB
            UserDaoImpl userDao = new UserDaoImpl(accessKey, secretKey, region, dynamoDbTableNamePrefix);
            
            m_logger.log("Get user record by username: " + username);
            User user = userDao.getUser(username);
            
            if(user != null)
            {
                // Add the model to the response map
                handlerResponse.getData().put("user", user);
                
                // Set our process status
                handlerResponse.setStatus(HandlerResponse.StatusKeys.SUCCESS);
                handlerResponse.setStatusMessage("");
            }
            else
            {
                throw new Exception("User record not found");
            }
        }
        catch (Exception e)
        {
            m_logger.log("Error processing request: " + e.getMessage());
            
            handlerResponse.setStatus(HandlerResponse.StatusKeys.FAILURE);
            handlerResponse.setStatusMessage(e.getMessage());
        }
        
        return handlerResponse;
    }
}
```

The JUnit class for testing the ViewUser Lambda function looks as follows:

```java
package com.nfbsoftware.sans_server.user.lambda;

import java.io.IOException;
import java.util.LinkedHashMap;

import org.junit.Assert;
import org.junit.BeforeClass;
import org.junit.Test;

import com.amazonaws.services.lambda.runtime.Context;
import com.nfbsoftware.sans_server.core.model.HandlerResponse;
import com.nfbsoftware.sans_server.junit.TestContext;
import com.nfbsoftware.sans_server.user.model.User;

/**
 * A simple test harness for locally invoking your Lambda function handler.
 * 
 * @author Brendan Clemenzi
 */
public class ViewUserTest
{
    private static LinkedHashMap<String, String> input;

    @BeforeClass
    public static void createInput() throws IOException
    {
        input = new LinkedHashMap<String, String>();
        input.put("username", "brendan@clemenzi.com");
    }

    private Context createContext()
    {
        TestContext ctx = new TestContext();

        ctx.setFunctionName("View User");

        return ctx;
    }

    @Test
    public void testViewUser()
    {
        ViewUser handler = new ViewUser();
        Context ctx = createContext();

        HandlerResponse output = (HandlerResponse)handler.handleRequest(input, ctx);

        if(output.getStatus().equalsIgnoreCase("SUCCESS")) 
        {
            User userValue = (User)output.getData().get("user");
            
            System.out.println("Full Name: " + userValue.getFullName());
            
            System.out.println("ViewUser JUnit Test Passed");
        }
        else
        {
            Assert.fail("ViewUser JUnit Test Failed");
        }
    }
}
```

