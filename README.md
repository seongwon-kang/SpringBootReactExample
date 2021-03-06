![travis-ci status](https://travis-ci.org/varren/SpringBootReactExample.svg?branch=master)

How this React SpringBoot project works


To start use`./gradlew bootRun`

Be careful with `yarn build` because it will remove all files in spring boot `src/main/resources/static/` directory
Can setup different path(in [gradle.build](https://github.com/varren/SpringBootReactExample/blob/master/build.gradle#L66) and [Config.java](https://github.com/varren/SpringBootReactExample/blob/master/src/main/java/ru/varren/Config.java#L43)) if you have some extra files you need to keep.



Implementation:
 
 1. Create Spring gradle spring-boot-starter-data-rest project
 
     **App.java** 
     
         package ru.varren;
         
         import org.springframework.boot.SpringApplication;
         import org.springframework.boot.autoconfigure.SpringBootApplication;
         
         @SpringBootApplication
         public class App {
             public static void main(String[] args) {
                 SpringApplication.run(App.class, args);
             }
         }
     
     **application.properties**
     
         spring.data.rest.base-path=/api
         
 2. Create react project
 
        npm install -g create-react-app
         
        create-react-app frontend
        cd frontend/
    1. Comment out `registerServiceWorker();` in `src/index.js` for now.
    
 3. Setup spring configuration
  
        @Configuration
        public class Config implements WebMvcConfigurer {
        
        
            @Override
            public void addResourceHandlers(ResourceHandlerRegistry registry) {
                ResourceResolver resolver = new ReactResourceResolver();
                registry.addResourceHandler("/**")
                        .resourceChain(true)
                        .addResolver(resolver);
            }
        
            public class ReactResourceResolver implements ResourceResolver {
                // this is the same directory you are using 
                // in package.json "build-spring-linux",
                // example REACT_DIR/index.html
                private static final String REACT_DIR = "/static/";
        
                // this is directory inside REACT_DIR for react static files
                // example REACT_DIR/REACT_STATIC_DIR/js/
                // example REACT_DIR/REACT_STATIC_DIR/css/
                private static final String REACT_STATIC_DIR = "static";
        
                private Resource index = new ClassPathResource(REACT_DIR + "index.html");
                private List<String> rootStaticFiles = Arrays.asList("favicon.io",
                        "asset-manifest.json", "manifest.json", "service-worker.js");
        
                @Override
                public Resource resolveResource(HttpServletRequest request, String requestPath,
                                                List<? extends Resource> locations, ResourceResolverChain chain) {
                    return resolve(requestPath, locations);
                }
        
                @Override
                public String resolveUrlPath(String resourcePath, List<? extends Resource> locations, ResourceResolverChain chain) {
                    Resource resolvedResource = resolve(resourcePath, locations);
                    if (resolvedResource == null) {
                        return null;
                    }
                    try {
                        return resolvedResource.getURL().toString();
                    } catch (IOException e) {
                        return resolvedResource.getFilename();
                    }
                }
        
                private Resource resolve(String requestPath, List<? extends Resource> locations) {
                    System.out.println(requestPath);
                    if (requestPath == null) return null;
        
                    if (rootStaticFiles.contains(requestPath)
                            || requestPath.startsWith(REACT_STATIC_DIR)) {
                        return new ClassPathResource(REACT_DIR + requestPath);
                    } else
                        return index;
                }
        
            }
        
        }

   4. Connecting `gradle:bootRun` with `yarn build-spring-linux`
        
        Add  `classpath "com.moowork.gradle:gradle-node-plugin:1.2.0"` gradle plugin

          node {
            
              // If true, it will download node using above parameters.
              // If false, it will try to use globally installed node.
              download = true
            
              // Set the work directory where node_modules should be located
              nodeModulesDir = file("${project.projectDir}/frontend")
          }

          // build react with webpack
          task webpack(type: YarnTask, dependsOn: yarn) {
              inputs.files(fileTree("${project.projectDir}/frontend/"))
              outputs.dir("${project.projectDir}/src/main/resources/static/")

              args = ['build']
          }

          // copy react build to spring resourses
          task moveJStoSpring(type: Copy) {
              from "${project.projectDir}/frontend/build/"
              into "${project.projectDir}/src/main/resources/static/"
          }

          moveJStoSpring.dependsOn 'webpack'
          processResources.dependsOn 'moveJStoSpring'

      Can comment out  `processResources.dependsOn 'webpack'` or implement some debug/production logic if you don't want gradle to recompile javascipt on each spring boot start
