# CloudSim on Windows: exact steps

## Folders used

Rename the zip folder if it differs.

```text
C:\cloudsim\                          <- extract the CloudSim 5.0 zip here
C:\cloudsim\commons-math3-3.6.1.jar   <- download the jar here
```

Jar download (use **math3 3.6.1**, the plain jar, not math4 and not the javadoc/sources jars):

```text
https://repo1.maven.org/maven2/org/apache/commons/commons-math3/3.6.1/commons-math3-3.6.1.jar
```

---

## Step 1: Create the project

1. Eclipse: **File → New → Java Project**.
2. Name: `CloudSim`.
3. Untick "Create module-info.java". Click **Finish**.
4. The project now has a folder called **`src`**. This is your **source folder**. Everything gets pasted here.

## Step 2: Copy the CloudSim code (copy 1)

1. In Windows File Explorer, open:
   ```text
   C:\cloudsim\cloudsim-cloudsim-5.0\modules\cloudsim\src\main\java
   ```
2. Inside you will see a folder named **`org`**. Right-click it → **Copy**.
3. In Eclipse, right-click **`src`** → **Paste**.

## Step 3: Copy the examples (copy 2)

1. In File Explorer, open:
   ```text
   C:\cloudsim\cloudsim-cloudsim-5.0\modules\cloudsim-examples\src\main\java
   ```
2. Copy the **`org`** folder again.
3. In Eclipse, right-click **`src`** → **Paste**. If it asks about merging, choose **Yes / Overwrite**.

## Step 4: Add the jar

1. Right-click the project → **Properties → Java Build Path → Libraries**.
2. Click **Classpath**, then **Add External JARs**.
3. Select `C:\cloudsim\commons-math3-3.6.1.jar`.
4. Click **Apply and Close**.

## Step 5: Run the test (Exp 11A)

1. In Eclipse, open **`src → org.cloudbus.cloudsim.examples`**.
2. Right-click **`CloudSimExample1.java`** → **Run As → Java Application**.
3. **Success:** the console ends with `CloudSimExample1 finished!`

## Step 6: FCFS program (Exp 11B)

1. Right-click the package **`org.cloudbus.cloudsim.examples`** → **New → Class**.
2. Name: `resourcealloc`. Click **Finish**. The first line `package org.cloudbus.cloudsim.examples;` appears automatically.
3. Delete everything **below** that package line, then paste the FCFS code from your sheet. Do **not** paste a second `package` line.
4. Change `CloudletSchedulerTimeShared` to `CloudletSchedulerSpaceShared` in the code and the import.
5. Right-click `resourcealloc.java` → **Run As → Java Application**.
6. **Success:** the console prints `FCFS RESOURCE ALLOCATION OUTPUT` with 8 cloudlets marked SUCCESS.

---

## If something is red

| Problem | Fix |
|---|---|
| Red errors in many files | Redo Step 4 (the jar) |
| `commons.math3 cannot be resolved` | The jar is not added; redo Step 4. Make sure it is `commons-math3-3.6.1.jar` |
| `Run As` shows no Java Application | Click the `.java` file first, then right-click it |
| Empty console | **Window → Show View → Console** |
| Folder `cloudsim-cloudsim-5.0` not found | Use the actual extracted folder name; the `modules` folder is inside it |
