# COBOL Hello World files for z/OS ADCD

Generated files:

- `HELLO.cbl` -> upload to `IBMUSER.COBOL.SOURCE(HELLO)`
- `HELLOC.jcl` -> upload to `IBMUSER.JCL(HELLOC)` or submit via JES
- `HELLOR.jcl` -> upload to `IBMUSER.JCL(HELLOR)` or submit via JES
- `ALLOC.jcl` -> optional allocation job if the data sets do not exist yet
- `ftp_commands.txt` -> suggested FTP command sequence

Notes:

- The compile JCL uses `IGY410.SIGYCOMP`, matching IBM COBOL for z/OS V4R1
  noted for this ADCD image.
- `HELLOC.jcl` writes the compiler object to a temporary `&&OBJ` data set and
  the load module to `IBMUSER.COBOL.LOAD(HELLO)`.
- `HELLOR.jcl` runs `PGM=HELLO` from `IBMUSER.COBOL.LOAD`.
- If `CEE.SCEELKED` is not cataloged or is already available through system
  defaults, adjust or remove the `SYSLIB` DD in `HELLOC.jcl`.
