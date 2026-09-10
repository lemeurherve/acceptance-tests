#!/usr/bin/env groovy

/*
 * Making sure that we can follow the steps necessary to install the latest
 * release on a Windows machine using the MSI installer.
 * See https://www.jenkins.io/doc/book/installing/windows/
 */

properties([
    buildDiscarder(logRotator(numToKeepStr: '15')),
    // Do not resume build after controller restart
    disableResume(),
    durabilityHint('PERFORMANCE_OPTIMIZED'),
    // Only one build running at a time, stop prior build if new build starts
    disableConcurrentBuilds(abortPrevious: true),
    pipelineTriggers([cron('H H/8 * * *')]), // Run once every 8 hours (three times a day)
])

// Define processors
def Processors = ['windows-2025']

// Generate a parallel step for each label in labels
def generateParallelSteps(labels) {
    def parallelNodes = [:]
    for (unboundLabel in labels) {
        def label = unboundLabel // Bind label before the closure
        parallelNodes[label] = {
            node(label) {
                stage('Download MSI') {
                    powershell 'Invoke-WebRequest -Uri https://get.jenkins.io/windows/latest/jenkins.msi -OutFile jenkins.msi'
                }


                stage('Install Jenkins from MSI') {
                    powershell '''
                        # Ensure the Windows Installer service is running before calling msiexec
                        $svc = Get-Service -Name msiserver -ErrorAction Stop
                        if ($svc.Status -ne "Running") {
                            Start-Service msiserver -ErrorAction Stop
                        }
                        $msi = Join-Path (Get-Location) "jenkins.msi"
                        $log = Join-Path (Get-Location) "jenkins-install.log"
                        $proc = Start-Process msiexec.exe `
                            -ArgumentList "/i `"$msi`" /qn /norestart /L*v `"$log`"" `
                            -Wait -PassThru
                        if ($proc.ExitCode -ne 0) {
                            if (Test-Path $log) { Get-Content $log | Select-Object -Last 50 }
                            throw "msiexec failed with exit code $($proc.ExitCode)"
                        }
                    '''
                }

                stage('Verify Jenkins service') {
                    powershell '''
                        $timeout = 60
                        $elapsed = 0
                        do {
                            $svc = Get-Service -Name Jenkins -ErrorAction SilentlyContinue
                            if ($svc -and $svc.Status -eq 'Running') { break }
                            Start-Sleep -Seconds 5
                            $elapsed += 5
                        } while ($elapsed -lt $timeout)
                        if (-not $svc -or $svc.Status -ne 'Running') {
                            throw "Jenkins service did not reach Running state within ${timeout}s (status: $($svc.Status))"
                        }
                        Write-Host "Jenkins service is Running"
                    '''
                }
            }
        }
    }
    return parallelNodes
}

timeout(unit: 'MINUTES', time:29) {
    stage('Processor') {
        parallel generateParallelSteps(Processors)
    }
}
