---
layout: post
comments: true
title:  "Run Cuckoo Generator manually using Fastlane"
date:   2019-04-30 15:08:00 +0800
categories: til programming
tags: [swift, programming, fastlane, cuckoo]
---
# Run Cuckoo Generator manually using Fastlane
## Problem
My team is using [Cuckoo generator](https://github.com/Brightify/Cuckoo) to generate mock classes for unit testing. Currently, I observe that it takes very long (80 seconds) to run a unit test after made a small change in the code.

## Root cause
I followed [Cuckoo'documentation](https://github.com/Brightify/Cuckoo#1-installation) when setting up for the project at first, by using the Run Script Build Phases. But when our code base getting bigger, it's easy to get code conflicts of .xcodeproj file when everyone keeps adding more input files to the build phase. So I decided to:
- Moved the content of the script inside the Build Phase to a separated `generate-mocks.sh` file and invoke that script file in the Build Phase. 

`generate-mocks.sh`
```
#!/bin/sh

# Define the output file. 
OUTPUT_FILE="$PROJECT_DIR/${PROJECT_NAME}Tests/GeneratedMocks.swift"
echo "Generated Mocks File = $OUTPUT_FILE"

INPUT_DIR="${PROJECT_DIR}/${PROJECT_NAME}"
echo "Mocks Input Directory = $INPUT_DIR"

# Generate mock files, include as many input files as you'd like to create mocks for.
"${PODS_ROOT}/Cuckoo/run" generate --testable "$PROJECT_NAME" \
--output "${OUTPUT_FILE}" \
"$INPUT_DIR/SampleProtocol.swift" \
"$INPUT_DIR/AnotherExampleProtocol.swift" \
.... more files here
```

- Removed all the Input Files of the Build Phase. 
  
By doing these, when we want to add more files to generate mocks, we just need to update `generate-mocks.sh` file, and avoid conflict in the scary *.xcodeproj* file.

But I didn't know that the Input Files section to help Xcode determine to skip the Build Phase if there is no change the Input Files list.

## Solutions
We can solve this by declaring all the files that need to generate mocks in Input Files of the Build Phase, but this will lead to easily get code conflict in .xcodeproj file.

We're already using [fastlane](https://fastlane.tools/), so I decided to use a lane to run Cuckoo Generator manually. The downside of this is, ya, you need to run it manually when needed.

Fastfile
```  
  desc "Generate Mocks using Cuckoo"
  lane :generate_mocks do 
    Dir.chdir('..') do
      project_dir = Dir.pwd
      project_name = "ShopBack"
      pods_root = Dir.pwd + "/Pods"
      sh("./scripts/cuckoo-generate-mocks.sh \"#{project_dir}\" \"#{project_name}\" \"#{pods_root}\"")
    end
  end
```

`generate-mocks.sh`
```
#!/bin/sh

# Read parameters
PROJECT_DIR="$1"
PROJECT_NAME="$2"
PODS_ROOT="$3"

# Define the output file. 
OUTPUT_FILE="$PROJECT_DIR/${PROJECT_NAME}Tests/GeneratedMocks.swift"
echo "Generated Mocks File = $OUTPUT_FILE"

INPUT_DIR="${PROJECT_DIR}/${PROJECT_NAME}"
echo "Mocks Input Directory = $INPUT_DIR"

# Generate mock files, include as many input files as you'd like to create mocks for.
"${PODS_ROOT}/Cuckoo/run" generate --testable "$PROJECT_NAME" \
--output "${OUTPUT_FILE}" \
"$INPUT_DIR/SampleProtocol.swift" \
"$INPUT_DIR/AnotherExampleProtocol.swift" \
.... more files here
```

## Result
This helped us save almost 80s waiting for unnecessary mocks generating, every time running a test case.